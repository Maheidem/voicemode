# VM-293 Technical Specification

## Overview

Add a "Voice Handoff Between Agents" section to the VoiceMode SKILL.md that documents how to transfer voice conversations between Claude Code agents in a multi-agent hierarchy (assistant → foreman → worker).

## Design Decisions

### 1. Placement in SKILL.md

**Decision**: Insert new section after "Common Workflows" (line 236) and before "Advanced Topics" (line 383).

**Rationale**: The handoff pattern is a common workflow for multi-agent setups. It belongs alongside other workflow patterns rather than in Advanced Topics (which covers provider internals and audio processing).

### 2. Section Structure

The new section will follow the existing SKILL.md patterns:
- Clear section header with explanation
- Concept introduction (hand-off vs hand-back)
- Step-by-step instructions with code examples
- Environment variable documentation
- Complete example scenario

### 3. Voice Configuration Documentation

**Decision**: Document `VOICEMODE_VOICES` environment variable.

**Format**: Comma-separated list of voice preferences (first for Kokoro, second for OpenAI).

**Role Defaults**:
- Foreman: `af_alloy,alloy` - professional/neutral tone
- Worker: Can inherit or use different voice for distinction
- Assistant: User's personal preference (already configured)

### 4. The TV Magazine Pattern Metaphor

Include the "TV Magazine Show" analogy from the task README - it makes the pattern intuitive:
- Host introduces segment
- Hands off to specialist reporter
- Viewer returns to host after segment

## Implementation Plan

### Step 1: Create New Section Content

Add the following section to SKILL.md (between lines 236 and 383):

```markdown
## Voice Handoff Between Agents

VoiceMode supports transferring voice conversations between agents in a multi-agent hierarchy. This is useful when an assistant spawns specialized agents (foreman, worker) to handle specific tasks.

### Hand-off vs Hand-back

| Term | Direction | Description |
|------|-----------|-------------|
| **Hand-off** | Down the chain | Transfer voice from assistant → foreman → worker |
| **Hand-back** | Up the chain | Return voice when receiving agent completes task |

Think of it like a TV magazine show: the host introduces a segment, hands off to a specialist reporter, then the viewer returns to the host after the segment ends.

### The Hand-off Pattern

When transferring voice to another agent:

1. **Announce the transfer** - Tell the user what's happening
2. **Spawn the agent** - Use `agents minion start` with voice context
3. **Go quiet** - Stop using converse tool (audio exclusivity)
4. **Monitor** - Wait for the spawned agent to complete
5. **Resume** - Take back the conversation when agent exits

**Critical Rule**: Only one agent should call `converse()` at a time. Audio is exclusive - competing agents cause confusion.

#### Example: Hand-off to Foreman

```python
# 1. Announce the transfer
voicemode:converse("I'm transferring you to the foreman to handle this task.", wait_for_response=False)

# 2. Spawn foreman with voice context
Bash(command="""VOICEMODE_VOICES=af_alloy,alloy agents minion start \\
  --role FOREMAN \\
  -p "You're being connected with the user via voice. They asked about [context]. \\
      Use the converse tool to speak with them. \\
      When finished, announce you're transferring back and stop conversing." \\
  /path/to/project""")

# 3. Go quiet - don't call converse() again until foreman exits
# 4. Monitor minion status (optional)
# 5. Resume when foreman exits
```

### Voice Configuration for Roles

Set `VOICEMODE_VOICES` when spawning agents to give each role a distinct voice:

```bash
# Format: kokoro_voice,openai_voice
VOICEMODE_VOICES=af_alloy,alloy agents minion start ...
```

**Recommended Voice Assignments:**

| Role | VOICEMODE_VOICES | Description |
|------|------------------|-------------|
| Foreman | `af_alloy,alloy` | Professional, neutral tone |
| Worker | `af_sky,fable` | (example) Distinct from foreman |
| Assistant | User preference | Already configured in user settings |

The first voice is used when Kokoro (local TTS) is available, the second for OpenAI TTS.

### The Hand-back Pattern

When the receiving agent finishes its task:

1. **Announce the return** - "I'm transferring you back to the assistant"
2. **Stop conversing** - Don't call converse() again
3. **Exit cleanly** - Complete the session normally

The calling agent detects completion by:
- Polling `agents minion status`
- Monitoring the minion process exit
- Reading minion output for completion signals

#### Example: Hand-back from Foreman

```python
# Foreman completes task, announces hand-back
voicemode:converse("Task complete. I'm transferring you back to the assistant.", wait_for_response=False)

# Foreman stops conversing and exits normally
# The calling assistant detects this and resumes the conversation
```

### Complete Example: Assistant → Foreman → Back

**Scenario**: User asks assistant to review a PR. Assistant hands off to foreman for detailed review.

**Assistant (initial conversation):**
```python
# User: "Can you review PR #123?"
voicemode:converse("I'll transfer you to my foreman to handle the PR review.", wait_for_response=False)

# Spawn foreman with voice and context
Bash(command="""VOICEMODE_VOICES=af_alloy,alloy agents minion start \\
  --role FOREMAN \\
  --name review-pr-123 \\
  -p "You're connected with the user via voice. They want you to review PR #123. \\
      Use converse() to speak with them throughout the review. \\
      When done, announce the handoff back and stop conversing." \\
  /path/to/repo""")

# Assistant goes quiet - waits for foreman to finish
```

**Foreman (handles the task):**
```python
# Foreman greets user
voicemode:converse("Hi, I'm the foreman. I'll review PR 123 for you.")

# [Performs review, discusses with user via converse()]

# When complete, hands back
voicemode:converse("Review complete. Transferring you back to the assistant.", wait_for_response=False)
# Foreman exits
```

**Assistant (resumes):**
```python
# Detects foreman exit, resumes conversation
voicemode:converse("I'm back. The foreman has completed the PR review. Is there anything else you need?")
```

### Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `VOICEMODE_VOICES` | Comma-separated voice preferences (kokoro,openai) | `af_alloy,alloy` |
| `AGENT_ROLE` | Current agent role (set by `--role` flag) | `FOREMAN` |

### Best Practices

1. **Audio exclusivity**: Never have two agents calling converse() simultaneously
2. **Clear announcements**: Always tell the user when transferring
3. **Distinct voices**: Use different voices for different roles so users can distinguish speakers
4. **Context passing**: Include relevant context in the spawn prompt
5. **Clean hand-backs**: Always announce before stopping conversations
6. **Monitor completion**: Use `agents minion status` to detect when spawned agents finish
```

### Step 2: Update File

Edit `plugins/voicemode/skills/voicemode/SKILL.md`:
- Insert new section after line 296 (end of "Checking and Troubleshooting Setup" subsection within Common Workflows)
- Before "Advanced Topics" section (current line 383)

### Step 3: Verify Changes

- [ ] Section appears in correct location
- [ ] All code examples are syntactically correct
- [ ] Tables render properly
- [ ] Links to related concepts work

## File Locations

| File | Purpose |
|------|---------|
| `worktree/plugins/voicemode/skills/voicemode/SKILL.md` | Target file for edits |
| `worktree/SPEC.md` | This specification |

## Acceptance Criteria Mapping

| Criterion | How Addressed |
|-----------|---------------|
| 1. VoiceMode SKILL.md has "Voice Handoff" section | New section added between Common Workflows and Advanced Topics |
| 2. Hand-off and hand-back distinguished | Separate subsections with step-by-step instructions |
| 3. VOICEMODE_VOICES documented with examples | Environment Variables table + role assignments table |
| 4. Complete example scenario provided | "Complete Example: Assistant → Foreman → Back" subsection |
| 5. Voice configuration per role documented | "Voice Configuration for Roles" subsection with recommended assignments |

## Testing Approach

Since this is a documentation task:
1. Render SKILL.md and verify markdown formatting
2. Validate all code blocks are syntactically plausible
3. Confirm section placement is logical within document flow
4. Check that all acceptance criteria are addressed
