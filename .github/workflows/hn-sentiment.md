---
# Trigger - respond to /hn-sentiment in issue comments only.
on:
  slash_command:
    name: hn-sentiment
    events: [issue_comment]

# Permissions - what can this workflow access?
# Write operations (creating issues, PRs, comments, etc.) are handled
# automatically by the safe-outputs job with its own scoped permissions.
permissions:
  contents: read
  issues: read
  pull-requests: read

# CAMBIO 1 - Engine: model that works with the Copilot Student plan
engine:
  id: copilot
  model: gpt-4.1

# CAMBIO 2 - Tools: bash lets the agent read the downloaded comments file
tools:
  bash: true

# Network access
network:
  allowed:
    - defaults
    - hacker-news.firebaseio.com

# Fetch the requested story and its first 20 top-level comments before analysis.
steps:
  - name: Fetch Hacker News comments
    env:
      COMMENT_BODY: ${{ github.event.comment.body }}
    run: |
      mkdir -p /tmp/gh-aw/agent
      url=$(printf '%s\n' "$COMMENT_BODY" | sed -nE 's#^/hn-sentiment[[:space:]]+(https://news\.ycombinator\.com/item\?id=[0-9]+)([[:space:]].*)?$#\1#p')
      if [ -z "$url" ]; then
        jq -n --arg error 'No valid Hacker News item URL was provided. Use /hn-sentiment https://news.ycombinator.com/item?id=12345.' '{error: $error}' > /tmp/gh-aw/agent/hn-sentiment.json
        exit 0
      fi

      item_id=$(printf '%s' "$url" | sed -nE 's#.*[?&]id=([0-9]+).*#\1#p')
      story=$(curl -fsS "https://hacker-news.firebaseio.com/v0/item/$item_id.json" || true)
      if [ -z "$story" ] || [ "$(printf '%s' "$story" | jq -r '.type // empty')" != "story" ]; then
        jq -n --arg error 'The URL does not identify a valid Hacker News story.' '{error: $error}' > /tmp/gh-aw/agent/hn-sentiment.json
        exit 0
      fi

      # CAMBIO 3 - Only 20 comments, keep just the text trimmed to 250 chars (fewer tokens, avoids 429)
      comment_ids=$(printf '%s' "$story" | jq -r '.kids[:20][]?')
      comments='[]'
      for comment_id in $comment_ids; do
        comment=$(curl -fsS "https://hacker-news.firebaseio.com/v0/item/$comment_id.json" || true)
        if [ -n "$comment" ]; then
          comments=$(jq -c --argjson comment "$comment" '. + [$comment | select(.type == "comment" and .dead != true and .deleted != true and .text != null) | {text: .text[:250]}]' <<< "$comments")
        fi
      done

      jq -n --arg url "$url" --argjson story "$story" --argjson comments "$comments" '{url: $url, story: {title: $story.title, id: $story.id}, comments: $comments}' > /tmp/gh-aw/agent/hn-sentiment.json

# Reply to the issue comment that triggered this workflow.
safe-outputs:
  add-comment:
    max: 1
    target: triggering
    footer: false
  # add-labels:
  # add-reviewer:
  # ado-assign-work-item:
  # ado-comment-on-work-item:
  # ado-create-work-item:
  # ado-link-work-items:
  # ado-update-work-item:
  # ado-upload-workitem-attachment:
  # allowed-github-references:
  # approve-workflow-run:
  # assign-milestone:
  # assign-to-agent:
  # assign-to-user:
  # autofix-code-scanning-alert:
  # call-workflow:
  # close-discussion:
  # close-issue:
  # close-pull-request:
  # concurrency-group:
  # create-agent-session:
  # create-agent-task:
  # create-check-run:
  # create-code-scanning-alert:
  # create-discussion:
  # create-project:
  # create-project-status-update:
  # create-pull-request:
  # create-pull-request-review-comment:
  # dismiss-pull-request-review:
  # dismiss-review:
  # dispatch-repository:
  # dispatch-workflow:
  # dispatch_repository:
  # environment:
  # failure-issue-repo:
  # group-reports:
  # hide-comment:
  # id-token:
  # jira-add-comment:
  # jira-add-label:
  # jira-create-issue:
  # jira-update-issue:
  # linear-add-comment:
  # linear-create-issue:
  # linear-token:
  # linear-update-issue:
  # link-sub-issue:
  # mark-pull-request-as-ready-for-review:
  # max-bot-mentions:
  # max-patch-files:
  # mentions:
  # merge-pull-request:
  # missing-data:
  # missing-tool:
  # noop:
  # push-to-pull-request-branch:
  # remove-labels:
  # replace-label:
  # reply-to-pull-request-review-comment:
  # report-failed-jobs:
  # report-failure-as-issue:
  # report-incomplete:
  # resolve-pull-request-review-thread:
  # scripts:
  # set-issue-field:
  # set-issue-type:
  # steer:
  # steps:
  # submit-pull-request-review:
  # threat-detection:
  # unassign-from-user:
  # update-discussion:
  # update-issue:
  # update-project:
  # update-pull-request:
  # update-release:
  # upload-artifact:
  # upload-asset:
  # upload-code-coverage:
  # urls:

---

# hn-sentiment

## Instructions

Read `/tmp/gh-aw/agent/hn-sentiment.json` once (a single command).

If the file contains `error`, reply to the triggering issue with that exact helpful error message and a valid command example. Do not perform sentiment analysis in this case.

Otherwise, analyze every comment in `comments` and classify its text as exactly one of Positive, Negative, or Neutral. Use the meaning and tone of the text, not the score or metadata. Rank comments by sentiment strength, using clear positive or negative language as stronger than mild language, and preserve the comment text only as short excerpts.

Reply to the triggering issue comment with one Markdown comment containing:

- A heading `## Hacker News Sentiment` and the story title and URL.
- The overall sentiment classification, chosen as the category with the largest count (use Neutral for a tie), and the percentage breakdown of Positive, Negative, and Neutral comments. Percentages must be based on all fetched comments and sum to 100% after rounding.
- A `### Most positive comments` section with up to 3 ranked items. Include each item's sentiment label and a concise excerpt; say `None found` when there are no comments.
- A `### Most negative comments` section with up to 3 ranked items. Include each item's sentiment label and a concise excerpt; say `None found` when there are no comments.

Do not include comments that were not fetched, do not invent comments, and keep excerpts brief. If the story has no usable comments, report that sentiment analysis is unavailable and show 0% for each category.

## Notes

- Run `gh aw compile` to generate the GitHub Actions workflow
- See https://github.github.com/gh-aw/ for complete configuration options and tools documentation