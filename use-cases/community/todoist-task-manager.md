# Todoist Task Manager: Agent Task Visibility
Maximize transparency for long-running agentic workflows by syncing internal reasoning and progress logs directly to Todoist.

## Pain Point
When agents run complex, multi-step tasks (like building a full-stack app or performing deep research), the user often loses track of what the agent is currently doing, what steps have been completed, and where the agent might be stuck. Checking chat logs manually is tedious for background tasks.

## What It Does
This use case uses the `todoist-task-manager` skill to:
1.  **Visualize State**: Create tasks in specific sections like `🟡 In Progress` or `🟠 Waiting`.
2.  **Externalize Reasoning**: Post the agent's internal "Plan" into the task description.
3.  **Stream Logs**: Add sub-step completions as comments to the task in real-time.
4.  **Auto-Reconcile**: A heartbeat script checks for stalled tasks and notifies the user.

## Skills you Need
You don't need a pre-built skill. Simply prompt your OpenClaw agent to create the bash scripts described in the **Setup Guide** below. Since OpenClaw can manage its own filesystem and execute shell commands, it will effectively "build" the skill for you upon request.

## Detailed Setup Guide

### 1. Configure Todoist
Create a project (e.g., "OpenClaw Workspace") and get its ID. Create sections for different states:
- `🟡 In Progress`
- `🟠 Waiting`
- `🟢 Done`

### 2. Implementation: The "Agent-Built" Skill
Instead of installing a skill, you can ask OpenClaw to create these scripts for you. Each script handles a different part of the communication with the Todoist API.

**`scripts/todoist_api.sh`** (The Core Wrapper):
```bash
#!/bin/bash
# Usage: ./todoist_api.sh <endpoint> <method> [data_json]
ENDPOINT=$1
METHOD=$2
DATA=$3
TOKEN="YOUR_TODOIST_API_TOKEN"

if [ -z "$DATA" ]; then
  curl -s -X "$METHOD" "https://api.todoist.com/rest/v2/$ENDPOINT" \
    -H "Authorization: Bearer $TOKEN"
else
  curl -s -X "$METHOD" "https://api.todoist.com/rest/v2/$ENDPOINT" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "$DATA"
fi
```

**`scripts/sync_task.sh`** (Task & Status Management):
```bash
#!/bin/bash
# Usage: ./sync_task.sh <task_content> <status> [task_id] [description] [labels_json_array]
CONTENT=$1
STATUS=$2
TASK_ID=$3
DESCRIPTION=$4
LABELS=$5
PROJECT_ID="YOUR_PROJECT_ID"

case $STATUS in
  "In Progress") SECTION_ID="SECTION_ID_PROGRESS" ;;
  "Waiting")     SECTION_ID="SECTION_ID_WAITING" ;;
  "Done")        SECTION_ID="SECTION_ID_DONE" ;;
  *)             SECTION_ID="" ;;
esac

PAYLOAD="{\"content\": \"$CONTENT\""
[ -n "$SECTION_ID" ] && PAYLOAD="$PAYLOAD, \"section_id\": \"$SECTION_ID\""
[ -n "$PROJECT_ID" ] && [ -z "$TASK_ID" ] && PAYLOAD="$PAYLOAD, \"project_id\": \"$PROJECT_ID\""
if [ -n "$DESCRIPTION" ]; then
  ESC_DESC=$(echo "$DESCRIPTION" | sed ':a;N;$!ba;s/\n/\\n/g' | sed 's/"/\\"/g')
  PAYLOAD="$PAYLOAD, \"description\": \"$ESC_DESC\""
fi
[ -n "$LABELS" ] && PAYLOAD="$PAYLOAD, \"labels\": $LABELS"
PAYLOAD="$PAYLOAD}"

if [ -n "$TASK_ID" ]; then
  ./scripts/todoist_api.sh "tasks/$TASK_ID" POST "$PAYLOAD"
else
  ./scripts/todoist_api.sh "tasks" POST "$PAYLOAD"
fi
```

**`scripts/add_comment.sh`** (Progress Logging):
```bash
#!/bin/bash
# Usage: ./add_comment.sh <task_id> <comment_text>
TASK_ID=$1
TEXT=$2
ESC_TEXT=$(echo "$TEXT" | sed ':a;N;$!ba;s/\n/\\n/g' | sed 's/"/\\"/g')
PAYLOAD="{\"task_id\": \"$TASK_ID\", \"content\": \"$ESC_TEXT\"}"
./scripts/todoist_api.sh "comments" POST "$PAYLOAD"
```

### 3. Usage Prompt
You can give this prompt to your agent to both **setup** and **use** the visibility system:

```text
I want you to build a Todoist-based task visibility system for your own runs. 

First, create three bash scripts in a 'scripts/' folder: 
1. todoist_api.sh (a curl wrapper for Todoist REST API)
2. sync_task.sh (to create or update tasks with specific section_ids for In Progress, Waiting, and Done)
3. add_comment.sh (to post progress logs as comments)

Use these variables for the setup:
- Token: [Your Todoist API Token]
- Project ID: [Your Project ID]
- Section IDs: [In Progress ID, Waiting ID, Done ID]

Once created, for every complex task I give you:
1. Create a task in 'In Progress' with your full PLAN in the description.
2. For every sub-step completion, call add_comment.sh with a log of what you did.
3. Move the task to 'Done' when finished.
```

## Related Links
- [Todoist REST API Documentation](https://developer.todoist.com/rest/v2/)

