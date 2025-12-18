# Task Decision Algorithm - Planning Document

## Overview

This document outlines the core decision algorithm for a task management service that uses OpenAI API to intelligently classify, relate, manage, estimate, and assign incoming tasks against existing tasks.

---

## 1. OpenAI API Analysis

### 1.1 Relevant API Features

#### Function Calling (Tool Use)
- **Best fit for this use case**
- Allows the model to call predefined functions to:
  - Fetch existing tasks
  - Fetch users by source for assignment
  - Get subtasks for migration
- Structured input/output ensures predictable behavior
- Supports parallel function calls for efficiency

```
Flow:
Input → OpenAI analyzes → Calls functions (tasks, users) → Makes decisions → Returns structured output
```

#### Structured Outputs (JSON Schema)
- Guarantees output matches predefined schema
- Essential for consistent downstream system updates
- Use `response_format: { type: "json_schema", json_schema: {...} }`

#### Key API Parameters to Consider
| Parameter | Recommended Value | Reason |
|-----------|------------------|--------|
| `model` | `gpt-4o` or `gpt-4-turbo` | Better reasoning for complex decisions |
| `temperature` | `0` or `0.1` | Deterministic decisions for consistency |
| `tool_choice` | `auto` or `required` | Ensure function calling when needed |

---

## 2. Complete Task Attributes

### 2.1 Task Object Schema

```json
{
  "id": "UUID",
  "title": "string (required)",
  "description": "string (required)",
  "status": "TODO | IN_PROGRESS | BLOCKED | IN_REVIEW | DONE | ARCHIVED",

  "timing": {
    "due_date": "ISO 8601 datetime | null",
    "estimated_minutes": "integer | null",
    "actual_minutes": "integer | null",
    "created_at": "ISO 8601 datetime",
    "updated_at": "ISO 8601 datetime",
    "started_at": "ISO 8601 datetime | null",
    "completed_at": "ISO 8601 datetime | null"
  },

  "assignment": {
    "assignee_id": "string | null",
    "assignee_name": "string | null",
    "reporter_id": "string | null",
    "reporter_name": "string | null",
    "watchers": ["array of user IDs"]
  },

  "classification": {
    "priority": "CRITICAL | HIGH | MEDIUM | LOW",
    "type": "FEATURE | BUG | TASK | EPIC | STORY | SPIKE",
    "labels": ["array of strings"],
    "project_id": "string | null",
    "sprint_id": "string | null"
  },

  "relationships": {
    "parent_id": "string | null",
    "subtask_ids": ["array of task IDs"],
    "blocked_by": ["array of task IDs"],
    "blocks": ["array of task IDs"],
    "related_to": ["array of task IDs"]
  },

  "source": {
    "source_id": "trello | teams | manual | other",
    "source_priority": "1-10",
    "external_id": "string | null",
    "external_url": "string | null",
    "raw_input": "string"
  },

  "metadata": {
    "version": "integer",
    "created_by_algorithm": "boolean",
    "confidence_score": "0.0-1.0",
    "requires_review": "boolean"
  }
}
```

### 2.2 Required vs Optional Attributes

| Attribute | Required | AI Extracted | Notes |
|-----------|----------|--------------|-------|
| title | Yes | Yes | Extracted from raw input |
| description | Yes | Yes | Extracted/summarized from input |
| status | Yes | No | Defaults to TODO |
| due_date | No | Yes | Extracted if mentioned in input |
| estimated_minutes | No | Yes | AI estimates based on task complexity |
| assignee_id | No | Yes | AI decides based on user list |
| priority | Yes | Yes | AI determines from context + source |
| type | Yes | Yes | AI classifies task type |

---

## 3. Input Structure

```json
{
  "source_id": "trello | teams | manual | other",
  "source_priority": 1-10,
  "task_content": {
    "raw_text": "string - the original input",
    "title": "string - extracted or provided title (optional)",
    "description": "string - extracted or provided description (optional)",
    "mentioned_users": ["array of user names/emails mentioned (optional)"],
    "mentioned_dates": ["array of date strings mentioned (optional)"],
    "metadata": {}
  },
  "context": {
    "timestamp": "ISO 8601",
    "user_id": "string (optional) - who submitted this",
    "project_id": "string (optional)",
    "channel_id": "string (optional) - for Teams"
  }
}
```

### Priority Rules
| Source | Default Priority | Notes |
|--------|-----------------|-------|
| Trello | 10 (highest) | Explicit task from task management system |
| Teams | 5 | May be task or conversation |
| Manual/Text | 3 | Requires classification first |
| Other | 1 | Lowest confidence |

---

## 4. Input Classification (Task vs Non-Task)

### 4.1 Classification Overview

**Not every incoming message should become a task.** Before proceeding with the full algorithm, the system must first determine if the input is:
- A valid, actionable task
- Something that should be ignored

### 4.2 Classification Categories

```
INPUT_CLASSIFICATION:

VALID_TASK:
  - Clear action or deliverable mentioned
  - Actionable by a team member
  - Has defined scope (even if broad)
  - Proceed to full algorithm

QUESTION_ONLY:
  - Seeking information, not action
  - No deliverable expected
  - Examples: "What's the status of X?", "Do we have Y?"
  - Action: IGNORE or RESPOND (not create task)

CONVERSATION:
  - General discussion or chat
  - Social/casual messages
  - Examples: "Thanks!", "Sounds good", "Let's discuss later"
  - Action: IGNORE

ACKNOWLEDGMENT:
  - Confirming receipt or agreement
  - No new action required
  - Examples: "Got it", "Will do", "Noted"
  - Action: IGNORE

ALREADY_TRACKED:
  - Refers to existing task without new info
  - Status update without action change
  - Examples: "Still working on X", "X is progressing"
  - Action: IGNORE (or UPDATE_STATUS if explicit)

INCOMPLETE:
  - Too vague to act on
  - Missing critical information
  - Examples: "Fix it", "Look into that thing"
  - Action: REQUEST_CLARIFICATION

SPAM/NOISE:
  - Automated messages, notifications
  - System-generated content
  - Action: IGNORE
```

### 4.3 Classification Algorithm

```
CLASSIFY_INPUT(raw_input, source):

1. CHECK source priority:
   - IF source == "trello":
     - High confidence it's a task
     - classification_bias = 0.8 toward VALID_TASK
   - ELSE:
     - No bias, evaluate content
     - classification_bias = 0.0

2. ANALYZE content signals:

   TASK_INDICATORS (positive signals):
   - Action verbs: "create", "fix", "implement", "update", "build", "review", "deploy"
   - Deliverable nouns: "feature", "bug", "report", "document", "API", "page"
   - Assignment language: "please", "need to", "should", "must", "can you"
   - Deadline mentions: "by Friday", "ASAP", "before release"
   - Explicit task markers: "TODO", "ACTION", "TASK:"

   NON_TASK_INDICATORS (negative signals):
   - Question-only: starts with "what", "why", "how", "is", "are", "did"
   - Social phrases: "thanks", "great", "sounds good", "👍", "ok"
   - Status without action: "still", "progressing", "ongoing"
   - Past tense completion: "done", "finished", "completed"
   - Conversational: "btw", "fyi", "just saying"

3. CALCULATE task_probability:

   task_signals = count(TASK_INDICATORS in raw_input)
   non_task_signals = count(NON_TASK_INDICATORS in raw_input)

   base_score = (task_signals - non_task_signals) / max(task_signals + non_task_signals, 1)
   adjusted_score = (base_score + classification_bias) / 2

   task_probability = normalize(adjusted_score, 0, 1)

4. DETERMINE classification:

   IF task_probability >= 0.7:
       classification = VALID_TASK
       action = PROCEED_TO_ALGORITHM

   ELIF task_probability >= 0.4:
       classification = UNCERTAIN
       action = FLAG_FOR_REVIEW
       # Could be task, human should verify

   ELIF has_question_pattern(raw_input):
       classification = QUESTION_ONLY
       action = IGNORE_OR_RESPOND

   ELIF length(raw_input) < 10 OR is_social_phrase(raw_input):
       classification = CONVERSATION
       action = IGNORE

   ELSE:
       classification = INCOMPLETE
       action = REQUEST_CLARIFICATION

5. RETURN:
   {
     "classification": "VALID_TASK | QUESTION_ONLY | CONVERSATION | ACKNOWLEDGMENT | INCOMPLETE | SPAM",
     "task_probability": 0.0-1.0,
     "action": "PROCEED_TO_ALGORITHM | IGNORE | REQUEST_CLARIFICATION | FLAG_FOR_REVIEW",
     "confidence": 0.0-1.0,
     "reasoning": "string explaining classification"
   }
```

### 4.4 Source-Specific Rules

| Source | Default Behavior | Notes |
|--------|------------------|-------|
| **Trello** | Assume VALID_TASK (0.9 probability) | Explicit task management system |
| **Teams** | Evaluate content (0.5 base) | Mix of chat and tasks |
| **Email** | Evaluate content (0.4 base) | Often conversational |
| **Manual** | Evaluate content (0.6 base) | User explicitly submitting |
| **Webhook** | Depends on webhook type | Configure per integration |

### 4.5 Examples

```
EXAMPLE 1:
Input: "We need to implement user authentication by next sprint"
Source: teams
Analysis:
  - Task indicators: "need to", "implement", "by next sprint"
  - No non-task indicators
  - task_probability: 0.85
Result: VALID_TASK → PROCEED_TO_ALGORITHM

EXAMPLE 2:
Input: "What's the status on the login bug?"
Source: teams
Analysis:
  - Starts with "What's"
  - Question pattern detected
  - task_probability: 0.2
Result: QUESTION_ONLY → IGNORE

EXAMPLE 3:
Input: "Fix it"
Source: manual
Analysis:
  - Task indicator: "fix"
  - But too vague, no context
  - Length < threshold
  - task_probability: 0.45
Result: INCOMPLETE → REQUEST_CLARIFICATION

EXAMPLE 4:
Input: "API Integration - Add OAuth support for third-party login"
Source: trello
Analysis:
  - Source is trello (high bias)
  - Clear task indicators
  - task_probability: 0.95
Result: VALID_TASK → PROCEED_TO_ALGORITHM

EXAMPLE 5:
Input: "Thanks for the update! 👍"
Source: teams
Analysis:
  - Social phrase: "Thanks"
  - Emoji
  - No task indicators
  - task_probability: 0.1
Result: CONVERSATION → IGNORE
```

### 4.6 Handling Uncertain Classifications

When `action = FLAG_FOR_REVIEW`:

```
UNCERTAIN_HANDLING:

1. Queue for human review
2. Include in response:
   - Original input
   - Classification reasoning
   - Suggested actions (proceed as task vs ignore)
   - Option to force proceed or dismiss

3. IF batch processing:
   - Collect uncertain items
   - Present for batch review
   - Learn from reviewer decisions (future improvement)

4. IF real-time:
   - Return clarification request to source
   - Example: "Is this a task you'd like me to track? Please confirm or provide more details."
```

---

## 5. Core Decision Algorithm

### 5.1 Algorithm Flow (Updated)

```
┌─────────────────────────────────────────────────────────────────┐
│                         INPUT RECEIVED                          │
│              (source_id, priority, task_content)                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│               STEP 0: INPUT CLASSIFICATION                      │
│         Is this a valid task or should it be ignored?           │
│         (See Section 4)                                         │
│                                                                 │
│         IF classification != VALID_TASK:                        │
│             → RETURN early (IGNORE/CLARIFY/REVIEW)              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼ (only if VALID_TASK)
┌─────────────────────────────────────────────────────────────────┐
│                    STEP 1: TASK EXTRACTION                      │
│         OpenAI extracts task intent from raw input              │
│         Extracts: title, description, mentioned dates/users     │
│         Output: normalized_task object                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 2: PARALLEL FUNCTION CALLS                    │
│    ┌──────────────────────┐    ┌──────────────────────┐        │
│    │  get_existing_tasks  │    │  get_users_by_source │        │
│    │  (keywords, filters) │    │  (source_id)         │        │
│    └──────────────────────┘    └──────────────────────┘        │
│         Returns: existing       Returns: available users        │
│         task objects            with ID, name, workload         │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 3: RELATIONSHIP ANALYSIS                    │
│         For each existing task, calculate:                      │
│         - Semantic similarity score (0-1)                       │
│         - Scope comparison (broader/narrower/equal)             │
│         - Priority comparison                                   │
│         - Due date conflict analysis                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STEP 4: DECISION MATRIX                       │
│              (Parent/Subtask/New Task Decision)                 │
│                    (See Section 4.2)                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 5: TIME ESTIMATION                          │
│         AI estimates task duration in minutes                   │
│         Based on: task complexity, similar tasks, scope         │
│                    (See Section 5)                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 STEP 6: DUE DATE ANALYSIS                       │
│         Determine/validate due date                             │
│         Check conflicts with related tasks                      │
│                    (See Section 6)                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 7: USER ASSIGNMENT                          │
│         Select best assignee from user list                     │
│         Based on: skills, workload, mentions                    │
│                    (See Section 7)                              │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  STEP 8: OUTPUT GENERATION                      │
│         Generate action object with all attributes              │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Decision Matrix (Task Relationship)

The algorithm uses these factors to make relationship decisions:

| Factor | Weight | Description |
|--------|--------|-------------|
| **Semantic Similarity** | 35% | How closely related is the incoming task to existing tasks |
| **Scope Comparison** | 30% | Is incoming task broader, narrower, or equal in scope |
| **Priority Delta** | 20% | Priority difference between incoming and existing |
| **Due Date Alignment** | 15% | How due dates relate to each other |

#### Decision Rules

```
RULE 1: NO RELATIONSHIP (similarity < 0.3)
├── Action: CREATE_NEW_TASK
└── No impact on existing tasks

RULE 2: SUBTASK CANDIDATE (similarity >= 0.3 AND scope = NARROWER)
├── Check: incoming_priority <= existing_priority
│   ├── TRUE:  Action: CREATE_AS_SUBTASK
│   │          └── Inherit parent's due_date if not specified
│   └── FALSE: Action: CREATE_NEW_TASK (flag for review)
└── Link to parent task

RULE 3: PARENT CANDIDATE (similarity >= 0.3 AND scope = BROADER)
├── Check: incoming_priority >= existing_priority
│   ├── TRUE:  Action: CREATE_AS_PARENT_AND_RETIRE
│   │          └── Trigger: SUBTASK_MIGRATION
│   │          └── Aggregate estimated_minutes from children
│   └── FALSE: Action: CREATE_NEW_TASK (flag for review)
└── Retire existing task(s)

RULE 4: DUPLICATE/CONFLICT (similarity >= 0.8 AND scope = EQUAL)
├── Check: incoming_priority > existing_priority
│   ├── TRUE:  Action: REPLACE_EXISTING
│   │          └── Preserve existing timing data if available
│   └── FALSE: Action: SKIP_OR_MERGE
└── Handle based on priority

RULE 5: PARTIAL OVERLAP (0.3 <= similarity < 0.8 AND scope = EQUAL)
├── Action: CREATE_NEW_TASK
├── Flag: POTENTIAL_CONFLICT (for human review)
└── Check: due_date conflicts → flag if overlapping
```

### 5.3 Scope Comparison Logic

```
SCOPE DETERMINATION:

Input: incoming_task, existing_task

1. Extract key entities/objectives from both tasks
2. Compare coverage:

   IF incoming covers ALL of existing + MORE:
       scope = BROADER

   ELSE IF existing covers ALL of incoming + MORE:
       scope = NARROWER

   ELSE IF significant overlap but neither fully contains other:
       scope = EQUAL

   ELSE:
       scope = UNRELATED
```

### 5.4 Subtask Migration Logic

When an incoming task becomes a PARENT and retires existing task(s):

```
SUBTASK MIGRATION ALGORITHM:

1. Get all subtasks of retired task(s)
2. For each subtask:
   a. Calculate relevance to new parent (similarity score)
   b. IF relevance >= 0.5:
      - Migrate subtask to new parent
      - Update subtask's parent_id
      - Preserve original estimated_minutes
   c. ELSE:
      - Flag for manual review
      - Optionally: create as independent task
3. Aggregate timing:
   - Sum estimated_minutes from all migrated subtasks
   - Earliest due_date becomes reference
4. Mark retired task as ARCHIVED (not deleted)
5. Create audit trail of migration
```

---

## 5. Time Estimation Algorithm

### 5.1 Estimation Factors

| Factor | Weight | Description |
|--------|--------|-------------|
| **Task Complexity** | 40% | Based on description analysis |
| **Similar Task History** | 30% | Average time of similar completed tasks |
| **Scope/Size** | 20% | Number of deliverables mentioned |
| **Type Baseline** | 10% | Default estimates by task type |

### 5.2 Complexity Classification

```
COMPLEXITY LEVELS:

TRIVIAL (15-30 minutes):
- Single, clear action
- No dependencies mentioned
- Keywords: "quick", "simple", "just", "minor"

LOW (30-120 minutes):
- 1-2 clear steps
- Minimal research needed
- Keywords: "update", "change", "fix typo"

MEDIUM (2-8 hours = 120-480 minutes):
- Multiple steps involved
- Some research/investigation needed
- Keywords: "implement", "create", "build", "investigate"

HIGH (1-3 days = 480-1440 minutes):
- Complex multi-step work
- Dependencies on other work
- Keywords: "design", "architect", "refactor", "major"

VERY_HIGH (3+ days = 1440+ minutes):
- Large feature/epic scope
- Multiple team members likely
- Keywords: "epic", "initiative", "overhaul", "migration"
```

### 5.3 Estimation Algorithm

```
ESTIMATE_MINUTES(task):

1. Determine complexity_level from description
2. Get base_estimate from complexity range midpoint
3. IF similar_tasks exist:
   - Get average actual_minutes from completed similar tasks
   - adjusted_estimate = (base_estimate * 0.4) + (similar_avg * 0.6)
4. ELSE:
   - adjusted_estimate = base_estimate
5. Apply type modifier:
   - BUG: * 1.2 (bugs often have hidden complexity)
   - SPIKE: * 0.8 (timeboxed by nature)
   - FEATURE: * 1.0 (baseline)
6. Round to nearest 15 minutes
7. Return estimated_minutes

OUTPUT: {
  "estimated_minutes": integer,
  "complexity": "TRIVIAL|LOW|MEDIUM|HIGH|VERY_HIGH",
  "confidence": 0.0-1.0,
  "reasoning": "string"
}
```

### 5.4 Estimation for Parent Tasks

```
PARENT TASK ESTIMATION:

IF task has subtasks:
   - Sum all subtask estimated_minutes
   - Add 10-20% buffer for coordination overhead
   - parent_estimated = sum(subtask_estimates) * 1.15

IF creating new parent from retired task:
   - Preserve retired task's actual_minutes if any
   - Re-estimate remaining work
   - Total = completed_work + remaining_estimate
```

---

## 6. Due Date Analysis Algorithm

### 6.1 Due Date Extraction

```
EXTRACT_DUE_DATE(raw_input):

1. Look for explicit date patterns:
   - "due by [date]", "deadline: [date]", "by [date]"
   - ISO dates, natural language dates

2. Look for relative dates:
   - "tomorrow", "next week", "end of sprint"
   - "ASAP", "urgent" → set to 24-48 hours

3. Look for contextual dates:
   - "before the release" → lookup release date
   - "Q1", "this quarter" → end of period

4. IF no date found AND task is subtask:
   - Inherit parent's due_date

5. IF no date found AND standalone:
   - Leave as null (to be set by assignee)

OUTPUT: {
  "due_date": "ISO 8601 | null",
  "due_date_source": "explicit | relative | inherited | inferred | none",
  "confidence": 0.0-1.0
}
```

### 6.2 Due Date Conflict Analysis

```
CHECK_DUE_DATE_CONFLICTS(task, related_tasks):

conflicts = []

FOR each related_task:

  IF task.due_date < related_task.due_date AND task blocks related_task:
      # Good - blocker is due before dependent
      CONTINUE

  IF task.due_date > related_task.due_date AND task blocks related_task:
      # Bad - blocker due after dependent
      conflicts.append({
        "type": "BLOCKER_AFTER_DEPENDENT",
        "task_id": related_task.id,
        "severity": "HIGH"
      })

  IF task is subtask AND task.due_date > parent.due_date:
      # Bad - subtask due after parent
      conflicts.append({
        "type": "SUBTASK_AFTER_PARENT",
        "task_id": parent.id,
        "severity": "MEDIUM",
        "suggestion": "Adjust subtask due date to " + parent.due_date
      })

  IF task.estimated_minutes > minutes_until(task.due_date):
      # Warning - not enough time
      conflicts.append({
        "type": "INSUFFICIENT_TIME",
        "severity": "HIGH",
        "estimated_minutes": task.estimated_minutes,
        "available_minutes": minutes_until(task.due_date)
      })

RETURN conflicts
```

### 6.3 Due Date Adjustment Logic

```
ADJUST_DUE_DATE(task, conflicts):

FOR each conflict:

  IF conflict.type == "SUBTASK_AFTER_PARENT":
      task.due_date = parent.due_date - buffer(1 day)
      task.flags.due_date_adjusted = true

  IF conflict.type == "INSUFFICIENT_TIME":
      IF task.due_date is flexible:
          new_date = now() + task.estimated_minutes + buffer
          task.due_date = new_date
          task.flags.due_date_adjusted = true
      ELSE:
          task.flags.requires_review = true
          task.flags.review_reason = "Insufficient time for deadline"

RETURN task
```

---

## 7. User Assignment Algorithm

### 7.1 User Data Structure (from get_users_by_source)

```json
{
  "users": [
    {
      "id": "string",
      "name": "string",
      "email": "string",
      "role": "developer | designer | qa | manager | etc",
      "skills": ["array of skill tags"],
      "current_workload": {
        "assigned_tasks": "integer",
        "total_estimated_minutes": "integer",
        "capacity_percentage": "0-100"
      },
      "availability": {
        "is_available": "boolean",
        "out_until": "ISO 8601 | null",
        "working_hours": "object"
      }
    }
  ]
}
```

### 7.2 Assignment Algorithm

```
ASSIGN_USER(task, users):

1. FILTER available users:
   - Remove users where is_available = false
   - Remove users where out_until > task.due_date

2. CHECK for explicit mentions:
   - IF task mentions user by name/email:
     - mentioned_user = find_user_by_mention(task.raw_input, users)
     - IF mentioned_user is available:
       - RETURN mentioned_user (confidence: 0.9)

3. SCORE remaining users:

   FOR each user:
     score = 0

     # Skill match (40%)
     skill_match = count_matching_skills(user.skills, task.labels)
     score += (skill_match / total_required_skills) * 40

     # Workload balance (35%)
     workload_score = (100 - user.capacity_percentage) / 100
     score += workload_score * 35

     # Role fit (15%)
     IF user.role matches task.type:
       score += 15

     # Past performance on similar tasks (10%)
     IF user has completed similar tasks:
       avg_performance = get_performance_score(user, similar_tasks)
       score += avg_performance * 10

     user.assignment_score = score

4. SELECT best user:
   - Sort users by assignment_score DESC
   - IF top_score > 50:
     - RETURN top_user (confidence: top_score/100)
   - ELSE:
     - RETURN null (flag for manual assignment)

OUTPUT: {
  "assignee_id": "string | null",
  "assignee_name": "string | null",
  "assignment_confidence": 0.0-1.0,
  "assignment_reasoning": "string",
  "alternative_assignees": [top 3 alternatives with scores]
}
```

### 7.3 Assignment Rules

```
SPECIAL ASSIGNMENT RULES:

1. SUBTASK INHERITANCE:
   - IF subtask AND parent has assignee:
     - Default to parent's assignee
     - Unless subtask requires different skill

2. BUG PRIORITY:
   - IF task.type == BUG AND priority == CRITICAL:
     - Prefer users with lowest current workload
     - Override skill matching weight to 20%
     - Increase workload weight to 55%

3. REASSIGNMENT ON PARENT CREATION:
   - IF creating parent from retired task:
     - Keep retired task's assignee for parent
     - Notify assignee of scope change

4. ROUND-ROBIN FALLBACK:
   - IF no clear winner (all scores < 30):
     - Use round-robin among available users
     - Track last_assigned_to for fairness
```

---

## 8. Output Structure

### 8.1 Complete Decision Output Schema

```json
{
  "classification": {
    "type": "VALID_TASK | QUESTION_ONLY | CONVERSATION | ACKNOWLEDGMENT | INCOMPLETE | SPAM",
    "task_probability": 0.0-1.0,
    "action_taken": "PROCEED_TO_ALGORITHM | IGNORE | REQUEST_CLARIFICATION | FLAG_FOR_REVIEW",
    "reasoning": "string - why this classification"
  },

  "decision": {
    "action": "CREATE_NEW_TASK | CREATE_AS_SUBTASK | CREATE_AS_PARENT_AND_RETIRE | REPLACE_EXISTING | SKIP_OR_MERGE | IGNORED | NEEDS_CLARIFICATION",
    "confidence": 0.0-1.0,
    "reasoning": "string - explanation of decision"
  },

  "task": {
    "id": "generated UUID",
    "title": "string",
    "description": "string",
    "status": "TODO",

    "timing": {
      "due_date": "ISO 8601 | null",
      "due_date_source": "explicit | relative | inherited | inferred | none",
      "estimated_minutes": "integer",
      "complexity": "TRIVIAL | LOW | MEDIUM | HIGH | VERY_HIGH"
    },

    "assignment": {
      "assignee_id": "string | null",
      "assignee_name": "string | null",
      "assignment_confidence": 0.0-1.0,
      "assignment_reasoning": "string",
      "alternative_assignees": []
    },

    "classification": {
      "priority": "CRITICAL | HIGH | MEDIUM | LOW",
      "type": "FEATURE | BUG | TASK | EPIC | STORY | SPIKE",
      "labels": []
    },

    "source": {
      "id": "original source_id",
      "priority": "number",
      "raw_input": "original input"
    }
  },

  "relationships": {
    "parent_id": "string | null",
    "is_parent_of": ["array of task IDs being retired"],
    "migrated_subtasks": ["array of subtask IDs"],
    "conflicts_with": ["array of task IDs with potential conflicts"]
  },

  "due_date_analysis": {
    "conflicts": [],
    "adjustments_made": [],
    "warnings": []
  },

  "flags": {
    "requires_review": true/false,
    "review_reasons": ["array of reasons"],
    "potential_duplicates": ["array of task IDs"],
    "due_date_adjusted": true/false,
    "assignment_uncertain": true/false
  },

  "actions": [
    {
      "type": "CREATE | UPDATE | ARCHIVE | LINK | UNLINK | ASSIGN | NOTIFY",
      "target_id": "task ID",
      "payload": {}
    }
  ],

  "audit": {
    "timestamp": "ISO 8601",
    "algorithm_version": "2.0",
    "model_used": "gpt-4o",
    "processing_time_ms": "number",
    "functions_called": ["list of function names"]
  }
}
```

### 8.2 Action Types

| Action Type | Description | Required Payload |
|-------------|-------------|------------------|
| `CREATE` | Create new task | full task object |
| `UPDATE` | Update existing task | task_id, fields to update |
| `ARCHIVE` | Retire/archive task | task_id, reason |
| `LINK` | Create parent-child relationship | parent_id, child_id |
| `UNLINK` | Remove relationship | parent_id, child_id |
| `ASSIGN` | Assign user to task | task_id, user_id |
| `NOTIFY` | Send notification | user_ids, message, type |

---

## 9. OpenAI Function Definitions

### 9.1 Complete Function Set

```json
{
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_existing_tasks",
        "description": "Fetch existing tasks from the system to compare against incoming task",
        "parameters": {
          "type": "object",
          "properties": {
            "keywords": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Keywords to search for related tasks"
            },
            "project_id": {
              "type": "string",
              "description": "Optional project filter"
            },
            "status_filter": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Task statuses to include (active, pending, etc.)"
            },
            "assignee_id": {
              "type": "string",
              "description": "Filter by assignee"
            },
            "due_date_range": {
              "type": "object",
              "properties": {
                "from": {"type": "string"},
                "to": {"type": "string"}
              }
            },
            "limit": {
              "type": "integer",
              "description": "Maximum number of tasks to return"
            }
          },
          "required": ["keywords"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_users_by_source",
        "description": "Fetch available users for task assignment based on the source system",
        "parameters": {
          "type": "object",
          "properties": {
            "source_id": {
              "type": "string",
              "description": "The source system (trello, teams, etc.)"
            },
            "include_workload": {
              "type": "boolean",
              "description": "Include current workload data"
            },
            "skills_filter": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Filter users by required skills"
            },
            "available_only": {
              "type": "boolean",
              "description": "Only return currently available users"
            }
          },
          "required": ["source_id"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_task_subtasks",
        "description": "Fetch subtasks of a specific task for migration analysis",
        "parameters": {
          "type": "object",
          "properties": {
            "task_id": {
              "type": "string",
              "description": "The parent task ID"
            },
            "include_completed": {
              "type": "boolean",
              "description": "Include completed subtasks"
            }
          },
          "required": ["task_id"]
        }
      }
    },
    {
      "type": "function",
      "function": {
        "name": "get_similar_completed_tasks",
        "description": "Fetch similar completed tasks for time estimation reference",
        "parameters": {
          "type": "object",
          "properties": {
            "keywords": {
              "type": "array",
              "items": {"type": "string"},
              "description": "Keywords describing the task"
            },
            "task_type": {
              "type": "string",
              "description": "Type of task (FEATURE, BUG, etc.)"
            },
            "limit": {
              "type": "integer",
              "description": "Maximum number of tasks to return"
            }
          },
          "required": ["keywords"]
        }
      }
    }
  ]
}
```

### 9.2 System Prompt Template

```
You are a Task Decision Engine. Your role is to analyze incoming tasks and:
1. Determine their relationship to existing tasks
2. Estimate time required
3. Analyze and set due dates
4. Assign to the most appropriate user

## Your Responsibilities:

### Step 1: Task Extraction
Extract from the raw input:
- Title (concise, action-oriented)
- Description (detailed requirements)
- Any mentioned dates or deadlines
- Any mentioned users
- Task type and priority indicators

### Step 2: Fetch Context (Parallel Calls)
- Call get_existing_tasks to find related tasks
- Call get_users_by_source to get available assignees

### Step 3: Relationship Analysis
For each existing task, determine:
- Semantic similarity (0-1)
- Scope comparison (BROADER/NARROWER/EQUAL)
- Priority comparison
- Due date alignment

### Step 4: Time Estimation
- Analyze task complexity
- Optionally call get_similar_completed_tasks for reference
- Estimate in minutes, round to nearest 15

### Step 5: Due Date Analysis
- Extract or infer due date
- Check for conflicts with related tasks
- Suggest adjustments if needed

### Step 6: User Assignment
- Score users based on skills, workload, mentions
- Select best fit or flag for manual assignment

## Decision Rules:
[Include full decision matrix]

## Output Requirements:
- Always return structured JSON matching the schema
- Be deterministic - same input should produce same output
- When uncertain (confidence < 0.7), set requires_review = true
- Provide clear reasoning for all decisions

## Priority Hierarchy:
Trello tasks (source_id: trello) have highest authority.
When conflicts arise, higher priority source wins.
```

---

## 10. Edge Cases & Handling

### 10.1 Classification Edge Cases

| Edge Case | Handling Strategy |
|-----------|-------------------|
| Empty input | Return SPAM classification, IGNORE |
| Single word input | Check if action verb → INCOMPLETE, else IGNORE |
| Only emojis | Return CONVERSATION, IGNORE |
| Mixed signals (task + question) | Higher weight to task signals if action verb present |
| Foreign language input | Attempt translation first, then classify |
| Code snippet only | Check context - could be task (fix this) or just sharing |
| URL only | Check URL type - Trello link = task reference, else IGNORE |
| Forwarded message | Analyze forwarded content, not "FW:" prefix |

### 10.2 Algorithm Edge Cases

| Edge Case | Handling Strategy |
|-----------|-------------------|
| No existing tasks found | CREATE_NEW_TASK with confidence: 1.0 |
| Multiple potential parents | Select highest priority, flag for review |
| Circular relationship detected | Reject, flag for manual resolution |
| Function call timeout | Retry once, then flag for async processing |
| Confidence below threshold | Always require human review |
| No available users | Flag for manual assignment, suggest wait |
| Past due date mentioned | Flag warning, suggest realistic date |
| Conflicting user mentions | List all mentioned, require clarification |
| Estimation too uncertain | Provide range instead of single value |
| Task references non-existent parent | Create as standalone, flag for review |

---

## 11. Algorithm Versioning

Maintain version in output for traceability:
- **v1.0**: Initial decision matrix with 3-factor weighting
- **v2.0**: Added time estimation, due date analysis, user assignment

---

## 12. Next Steps

- [ ] Review and refine decision matrix thresholds
- [ ] Define exact database schema for tasks
- [ ] Implement function handlers
- [ ] Build test cases for each decision path
- [ ] Create monitoring for decision quality
- [ ] Define user skill taxonomy
- [ ] Set up workload calculation service
- [ ] Create feedback loop for estimation accuracy
