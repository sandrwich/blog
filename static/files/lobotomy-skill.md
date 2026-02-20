---
name: lobotomy
description: Selective context amnesia - inject a fake compact_boundary to drop old conversation history while preserving recent context and vibe.
disable-model-invocation: true
argument-hint: [what to forget]
allowed-tools:
  - Bash
  - Read
  - Write
---

# Lobotomy Procedure

The user wants to perform selective context amnesia by injecting a synthetic `compact_boundary` into the conversation JSONL.

## Arguments

`$ARGUMENTS` is a natural language description of what to forget. Examples:
- `"forget everything before we talked about the API"`
- `"cut before the debugging session"`
- `"keep only the last 20 messages"`
- `"forget the first half of the conversation"`
- Empty = interactive mode (show recent messages, let user pick)

## Procedure

### Step 1: Locate the conversation file

The current session ID is `${CLAUDE_SESSION_ID}`. Find the JSONL file:

```bash
ls -la ~/.claude/projects/*/$(echo "${CLAUDE_SESSION_ID}").jsonl 2>/dev/null || \
ls -lt ~/.claude/projects/*/*.jsonl | head -5
```

### Step 2: Backup

**ALWAYS** create a backup before modifying:

```bash
cp <session>.jsonl <session>.jsonl.bak
```

### Step 3: Find CLEAN cut point

**CRITICAL: Cut points must not orphan tool_use/tool_result pairs.** If a `tool_result` after the cut references a `tool_use` before the cut, the API will reject the conversation with a 400 error.

Use this script to find **only clean** cut points (searches only after the latest boundary):

```python
import json

with open('<session>.jsonl') as f:
    lines = f.readlines()

# Find latest boundary to avoid showing ancient pre-boundary messages
last_boundary = 0
for i, line in enumerate(lines):
    obj = json.loads(line)
    if obj.get('subtype') == 'compact_boundary':
        last_boundary = i

print(f"Latest boundary at L{last_boundary}")

# Index tool_use/tool_result references (only after boundary)
tool_uses = {}   # tool_use_id -> line_number
tool_results = {} # tool_use_id -> line_number

for i in range(last_boundary, len(lines)):
    obj = json.loads(lines[i])
    msg = obj.get('message', {})
    content = msg.get('content', [])
    if isinstance(content, list):
        for block in content:
            if isinstance(block, dict):
                if block.get('type') == 'tool_use':
                    tool_uses[block.get('id', '')] = i
                elif block.get('type') == 'tool_result':
                    tool_results[block.get('tool_use_id', '')] = i

# Find clean user messages (no orphaned references if we cut here)
for i in range(last_boundary, len(lines)):
    obj = json.loads(lines[i])
    if obj.get('type') != 'user':
        continue
    msg = obj.get('message', {})
    content = msg.get('content', '')
    if not isinstance(content, str) or len(content) < 5:
        continue

    # Check: would any tool_result after this line reference a tool_use before it?
    orphans = False
    for tu_id, result_line in tool_results.items():
        if result_line >= i:
            use_line = tool_uses.get(tu_id)
            if use_line is not None and use_line < i:
                orphans = True
                break

    if not orphans:
        ts = obj.get('timestamp', '')[:19]
        preview = content[:80].replace('\n', ' ')
        print(f"CLEAN L{i} @ {ts}: {preview}")
```

Based on the user's description (`$ARGUMENTS`), identify the appropriate **clean** line number. If `$ARGUMENTS` is empty, show the last ~50 clean user messages and ask the user to pick.

**Always confirm with the user before proceeding:** "I'll cut at line X, which is right before '[message preview]'. Everything before this will be forgotten. Proceed?"

### Step 4: Memory Dump (CRITICAL)

**Before cutting, output a condensed memory summary directly to chat.** This is your last chance to preserve context. The user can copy this if needed.

Output a summary that includes:
- Key topics discussed and conclusions reached
- Important technical findings or decisions
- User preferences learned
- Relationship context (tone, inside jokes, rapport)
- Anything critical the post-lobotomy you should know

**Output format (directly to chat, no file):**

```
# Pre-Lobotomy Memory Dump
Cut point: Line <N> - "<first few words of message>"

## Key Context
<your condensed summary here>

## Topics Covered
- <topic 1>
- <topic 2>

## Important Findings
- <finding 1>

## Vibe Notes
<tone, rapport, inside jokes>

## Critical Information
<anything the post-lobotomy you MUST know>
```

The boundary injection script (Step 6) will also include this summary as a message in the JSONL so it's in-context after restart.

### Step 5: Get UUID of message before cut point

Walk backwards from the cut line to find the nearest line with a UUID (some lines like file-history-snapshots may not have one):

```python
import json
with open('<session>.jsonl') as f:
    lines = f.readlines()
    cut_line = <CUT_LINE_NUMBER>
    for i in range(cut_line - 1, max(0, cut_line - 20), -1):
        obj = json.loads(lines[i])
        if 'uuid' in obj:
            print(f"Line {i}: uuid={obj['uuid']}")
            print(f"Type: {obj['type']}")
            break
```

Use this UUID in Step 6 as `PARENT_UUID`.

### Step 6: Inject boundary with memory summary

Use this Python script (fill in the values, including your MEMORY_SUMMARY):

```python
import json, uuid
from datetime import datetime, timezone

JSONL_PATH = '<session>.jsonl'
CUT_LINE = <LINE_NUMBER>
PARENT_UUID = '<uuid-from-step-5>'

# Your memory summary from Step 4 - paste the condensed version here
MEMORY_SUMMARY = '''
<paste your memory dump summary here>
'''

with open(JSONL_PATH) as f:
    lines = f.readlines()

# Find an existing boundary to copy structure from
template = None
for line in lines:
    obj = json.loads(line)
    if obj.get('subtype') == 'compact_boundary':
        template = obj
        break

if not template:
    print("ERROR: No existing boundary to copy from")
    exit(1)

boundary_uuid = str(uuid.uuid4())
summary_uuid = str(uuid.uuid4())

boundary = {
    'parentUuid': PARENT_UUID,
    'logicalParentUuid': PARENT_UUID,
    'isSidechain': False,
    'userType': 'external',
    'cwd': template.get('cwd'),
    'sessionId': template.get('sessionId'),
    'version': template.get('version'),
    'gitBranch': 'HEAD',
    'slug': template.get('slug', ''),
    'type': 'system',
    'subtype': 'compact_boundary',
    'content': 'Conversation compacted',
    'isMeta': False,
    'timestamp': datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z'),
    'uuid': boundary_uuid,
    'level': template.get('level'),
    'compactMetadata': {
        'trigger': 'manual',
        'preTokens': 170000
    },
    'forkedFrom': template.get('forkedFrom')
}

# Create a summary message that will appear right after the boundary
summary_msg = {
    'parentUuid': boundary_uuid,
    'isSidechain': False,
    'userType': 'external',
    'cwd': template.get('cwd'),
    'sessionId': template.get('sessionId'),
    'version': template.get('version'),
    'gitBranch': 'HEAD',
    'slug': template.get('slug', ''),
    'type': 'user',
    'message': {
        'role': 'user',
        'content': f'[LOBOTOMY MEMORY DUMP - Read this to understand context before the cut]\n\n{MEMORY_SUMMARY}'
    },
    'uuid': summary_uuid,
    'timestamp': datetime.now(timezone.utc).isoformat().replace('+00:00', 'Z'),
}

# Update next message's parentUuid to point to summary (not boundary)
next_msg = json.loads(lines[CUT_LINE])
next_msg['parentUuid'] = summary_uuid

# Write back
with open(JSONL_PATH, 'w') as f:
    for i, line in enumerate(lines):
        if i == CUT_LINE:
            f.write(json.dumps(boundary) + '\n')
            f.write(json.dumps(summary_msg) + '\n')
            f.write(json.dumps(next_msg) + '\n')
        else:
            f.write(line)

print(f"Lobotomy complete with memory dump.")
print(f"Boundary UUID: {boundary_uuid}")
print(f"Summary UUID: {summary_uuid}")
print("RESTART THE CLI for changes to take effect.")
```

### Step 7: Instruct user

After executing the script, tell the user:

> **Lobotomy complete with memory preservation.**
>
> The synthetic boundary and memory dump have been inserted. To activate:
> 1. Exit this CLI session (Ctrl+C or `/exit`)
> 2. Restart Claude Code
>
> When I return:
> - I will have no direct memory of anything before line {CUT_LINE}
> - BUT the memory dump will be in-context as the first message after the boundary
> - I should read it to understand what came before
>
> **To undo (if something goes wrong):**
> ```bash
> cp <session>.jsonl.bak <session>.jsonl
> ```
>
> **To permanently delete the backup (once you're happy):**
> ```bash
> rm <session>.jsonl.bak
> ```

Replace `<session>` with the actual file path in both commands.

## Important Notes

- The lobotomy does NOT take effect until CLI restart
- Vibe/style should persist because recent messages are preserved
- Search through user messages to find the right cut point based on `$ARGUMENTS`
- Always confirm the cut point with the user before executing
- This procedure was documented in paper2.pdf by Sandra & Claude Opus 4.6 (2026)
