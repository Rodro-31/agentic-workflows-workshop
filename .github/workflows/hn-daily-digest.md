---
name: HN Daily Digest
# Trigger - when should this workflow run?
on:
  schedule: daily on weekdays
  workflow_dispatch:  # Manual trigger

# Alternative triggers (uncomment to use):
# on:
#   issues:
#     types: [opened, reopened]
#   pull_request:
#     types: [opened, synchronize]
#   schedule: daily  # Fuzzy daily schedule (scattered execution time)
#   # schedule: weekly on monday  # Fuzzy weekly schedule

# Permissions - what can this workflow access?
# Write operations (creating issues, PRs, comments, etc.) are handled
# automatically by the safe-outputs job with its own scoped permissions.
permissions:
  contents: read
  issues: read
  pull-requests: read

engine:
  id: copilot
  model: gpt-4.1

# Tools - bash lets the agent read the downloaded HN data file
tools:
  bash: true
#   github:
#     toolsets: [default]

# Network access
network:
  allowed:
    - defaults
    - hacker-news.firebaseio.com

# Steps - run before the agent: download HN top stories to a JSON file
steps:
  - name: Fetch HN top stories
    run: |
      mkdir -p /tmp/gh-aw/agent
      for id in $(curl -s https://hacker-news.firebaseio.com/v0/topstories.json | jq -r '.[:30][]'); do
        curl -s "https://hacker-news.firebaseio.com/v0/item/$id.json"
      done | jq -s '[.[] | select(.score > 100) | {title, url, score, comments: .descendants}]' > /tmp/gh-aw/agent/hn.json

# Outputs - what APIs and tools can the AI use?
safe-outputs:
  create-issue:          # Creates issues (default max: 1)
    max: 1               # One digest per run
  # actions:
  # activation-comments:
  # add-comment:
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

# HN Daily Digest

## Instructions

Read the file /tmp/gh-aw/agent/hn.json. It contains the top Hacker News stories with score above 100 (title, url, score, comments).

Keep only stories about software engineering, cloud infrastructure, AI/ML, developer tooling, or distributed systems that are useful today for large-company developers.

Create one GitHub issue titled "HN Digest - <today's date>" with a Markdown table with columns: Title, URL, Score, Comments, Why it matters (one sentence for enterprise developers).

If no story qualifies, say so in the issue.

## Notes

- Run `gh aw compile` to generate the GitHub Actions workflow
- See https://github.github.com/gh-aw/ for complete configuration options and tools documentation