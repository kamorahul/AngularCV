# Task Decision Algorithm - Planning Document

## Overview

This document outlines the core decision algorithm for a task management service that uses OpenAI API to intelligently classify, relate, and manage incoming tasks against existing tasks.

---

## 1. OpenAI API Analysis

### 1.1 Relevant API Features

#### Function Calling (Tool Use)
- **Best fit for this use case**
- Allows the model to call predefined functions to fetch existing tasks
- Structured input/output ensures predictable behavior
- Supports parallel function calls for efficiency

```
Flow:
Input → OpenAI analyzes → Calls function to fetch existing tasks → Makes decision → Returns structured output
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

## 2. Input Structure

```json
{
  "source_id": "trello|teams|manual|other",
  "source_priority": 1-10,
  "task_content": {
    "raw_text": "string - the original input",
    "title": "string - extracted or provided title (optional)",
    "description": "string - extracted or provided description (optional)",
    "metadata": {}
  },
  "context": {
    "timestamp": "ISO 8601",
    "user_id": "string (optional)",
    "project_id": "string (optional)"
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

## 3. Core Decision Algorithm

### 3.1 Algorithm Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                         INPUT RECEIVED                          │
│              (source_id, priority, task_content)                │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                    STEP 1: TASK EXTRACTION                      │
│         OpenAI extracts task intent from raw input              │
│         Output: normalized_task object                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                 STEP 2: FETCH EXISTING TASKS                    │
│         Function call: get_existing_tasks(filters)              │
│         Returns: array of existing task objects                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 3: RELATIONSHIP ANALYSIS                    │
│         For each existing task, calculate:                      │
│         - Semantic similarity score (0-1)                       │
│         - Scope comparison (broader/narrower/equal)             │
│         - Priority comparison                                   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STEP 4: DECISION MATRIX                       │
│                    (See Section 3.2)                            │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  STEP 5: OUTPUT GENERATION                      │
│         Generate action object with predefined attributes       │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Decision Matrix

The algorithm uses three key factors to make decisions:

| Factor | Weight | Description |
|--------|--------|-------------|
| **Semantic Similarity** | 40% | How closely related is the incoming task to existing tasks |
| **Scope Comparison** | 35% | Is incoming task broader, narrower, or equal in scope |
| **Priority Delta** | 25% | Priority difference between incoming and existing |

#### Decision Rules

```
RULE 1: NO RELATIONSHIP (similarity < 0.3)
├── Action: CREATE_NEW_TASK
└── No impact on existing tasks

RULE 2: SUBTASK CANDIDATE (similarity >= 0.3 AND scope = NARROWER)
├── Check: incoming_priority <= existing_priority
│   ├── TRUE:  Action: CREATE_AS_SUBTASK
│   └── FALSE: Action: CREATE_NEW_TASK (flag for review)
└── Link to parent task

RULE 3: PARENT CANDIDATE (similarity >= 0.3 AND scope = BROADER)
├── Check: incoming_priority >= existing_priority
│   ├── TRUE:  Action: CREATE_AS_PARENT_AND_RETIRE
│   │          └── Trigger: SUBTASK_MIGRATION
│   └── FALSE: Action: CREATE_NEW_TASK (flag for review)
└── Retire existing task(s)

RULE 4: DUPLICATE/CONFLICT (similarity >= 0.8 AND scope = EQUAL)
├── Check: incoming_priority > existing_priority
│   ├── TRUE:  Action: REPLACE_EXISTING
│   └── FALSE: Action: SKIP_OR_MERGE
└── Handle based on priority

RULE 5: PARTIAL OVERLAP (0.3 <= similarity < 0.8 AND scope = EQUAL)
├── Action: CREATE_NEW_TASK
└── Flag: POTENTIAL_CONFLICT (for human review)
```

### 3.3 Scope Comparison Logic

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

### 3.4 Subtask Migration Logic

When an incoming task becomes a PARENT and retires existing task(s):

```
SUBTASK MIGRATION ALGORITHM:

1. Get all subtasks of retired task(s)
2. For each subtask:
   a. Calculate relevance to new parent (similarity score)
   b. IF relevance >= 0.5:
      - Migrate subtask to new parent
      - Update subtask's parent_id
   c. ELSE:
      - Flag for manual review
      - Optionally: create as independent task
3. Mark retired task as ARCHIVED (not deleted)
4. Create audit trail of migration
```

---

## 4. Output Structure

### 4.1 Decision Output Schema

```json
{
  "decision": {
    "action": "CREATE_NEW_TASK | CREATE_AS_SUBTASK | CREATE_AS_PARENT_AND_RETIRE | REPLACE_EXISTING | SKIP_OR_MERGE",
    "confidence": 0.0-1.0,
    "reasoning": "string - explanation of decision"
  },
  "task": {
    "id": "generated UUID",
    "title": "string",
    "description": "string",
    "normalized_content": "string",
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
  "flags": {
    "requires_review": true/false,
    "review_reason": "string | null",
    "potential_duplicates": ["array of task IDs"]
  },
  "actions": [
    {
      "type": "CREATE | UPDATE | ARCHIVE | LINK | UNLINK",
      "target_id": "task ID",
      "payload": {}
    }
  ],
  "audit": {
    "timestamp": "ISO 8601",
    "algorithm_version": "1.0",
    "model_used": "gpt-4o",
    "processing_time_ms": "number"
  }
}
```

### 4.2 Action Types

| Action Type | Description | Required Payload |
|-------------|-------------|------------------|
| `CREATE` | Create new task | task object |
| `UPDATE` | Update existing task | task_id, fields to update |
| `ARCHIVE` | Retire/archive task | task_id, reason |
| `LINK` | Create parent-child relationship | parent_id, child_id |
| `UNLINK` | Remove relationship | parent_id, child_id |

---

## 5. OpenAI Implementation Strategy

### 5.1 Function Definitions

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
        "name": "get_task_subtasks",
        "description": "Fetch subtasks of a specific task for migration analysis",
        "parameters": {
          "type": "object",
          "properties": {
            "task_id": {
              "type": "string",
              "description": "The parent task ID"
            }
          },
          "required": ["task_id"]
        }
      }
    }
  ]
}
```

### 5.2 System Prompt Template

```
You are a Task Decision Engine. Your role is to analyze incoming tasks and determine their relationship to existing tasks in the system.

## Your Responsibilities:
1. Extract the core intent/objective from the incoming task
2. Call get_existing_tasks to fetch potentially related tasks
3. Analyze relationships using these criteria:
   - Semantic similarity (how related is the content)
   - Scope comparison (is one task broader/narrower than another)
   - Priority comparison (which task takes precedence)

## Decision Rules:
[Include decision matrix rules here]

## Output Requirements:
Always return a structured decision following the exact JSON schema provided.
Be deterministic - same input should produce same output.
When uncertain (confidence < 0.7), set requires_review = true.

## Priority Hierarchy:
Trello tasks (source_id: trello) have highest authority.
When conflicts arise, higher priority source wins.
```

### 5.3 Conversation Flow

```
Message 1 (User/System):
{
  "role": "user",
  "content": "[JSON input with source_id, priority, task_content]"
}

Message 2 (Assistant):
→ Calls get_existing_tasks function

Message 3 (Tool Response):
{
  "role": "tool",
  "content": "[Array of existing tasks]"
}

Message 4 (Assistant):
→ If needed, calls get_task_subtasks for retirement analysis

Message 5 (Tool Response):
{
  "role": "tool",
  "content": "[Subtasks array]"
}

Message 6 (Assistant):
→ Returns final decision JSON
```

---

## 6. Edge Cases & Handling

| Edge Case | Handling Strategy |
|-----------|-------------------|
| No existing tasks found | CREATE_NEW_TASK with confidence: 1.0 |
| Multiple potential parents | Select highest priority, flag for review |
| Circular relationship detected | Reject, flag for manual resolution |
| Empty/invalid input | Return error with validation details |
| Function call timeout | Retry once, then flag for async processing |
| Confidence below threshold | Always require human review |

---

## 7. Algorithm Versioning

Maintain version in output for traceability:
- **v1.0**: Initial decision matrix with 3-factor weighting
- Future versions may adjust weights based on feedback

---

## 8. Open Questions / Future Considerations

1. **Threshold Tuning**: Similarity thresholds (0.3, 0.8) may need adjustment based on real-world data
2. **Multi-language Support**: If tasks come in different languages, translation step needed
3. **Batch Processing**: Algorithm designed for single task; batch optimization possible
4. **Learning Loop**: Could add feedback mechanism to improve decisions over time
5. **Conflict Resolution UI**: When requires_review = true, what's the human interface?

---

## Next Steps

- [ ] Review and refine decision matrix thresholds
- [ ] Define exact database schema for tasks
- [ ] Implement function handlers for get_existing_tasks
- [ ] Build test cases for each decision path
- [ ] Create monitoring for decision quality
