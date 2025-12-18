# Task Decision System - Architecture

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                   INPUT SOURCES                                      │
├──────────────┬──────────────┬──────────────┬──────────────┬────────────────────────┤
│    Trello    │    Teams     │    Email     │   Manual     │      Webhooks          │
│   (API/WH)   │  (Bot/API)   │  (Parser)    │   (UI/API)   │    (Custom)            │
└──────┬───────┴──────┬───────┴──────┬───────┴──────┬───────┴───────────┬────────────┘
       │              │              │              │                   │
       └──────────────┴──────────────┴──────────────┴───────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              INPUT NORMALIZER                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ Source Adapter  │  │ Schema Mapper   │  │ Sender Enricher │                      │
│  │ (per source)    │  │ (normalize)     │  │ (role lookup)   │                      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           TASK DECISION ENGINE                                       │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                         OpenAI GPT-4 Integration                              │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                  │   │
│  │  │ Classification │  │ Relationship   │  │ Estimation     │                  │   │
│  │  │ & Role Weight  │  │ Analysis       │  │ & Assignment   │                  │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘                  │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                     │                                                │
│                    ┌────────────────┼────────────────┐                              │
│                    ▼                ▼                ▼                              │
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐           │
│  │ Function Handlers   │ │ Function Handlers   │ │ Function Handlers   │           │
│  │ get_existing_tasks  │ │ get_users_by_source │ │ get_similar_tasks   │           │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                            ACTION DISPATCHER                                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │
│  │   CREATE    │ │   UPDATE    │ │   COMMENT   │ │   ASSIGN    │ │   NOTIFY    │   │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              OUTPUT TARGETS                                          │
├──────────────┬──────────────┬──────────────┬──────────────┬────────────────────────┤
│   Task DB    │   Trello     │    Jira      │  Slack/Teams │    Audit Log           │
│  (Primary)   │   (Sync)     │   (Sync)     │ (Notify)     │   (History)            │
└──────────────┴──────────────┴──────────────┴──────────────┴────────────────────────┘
```

---

## 2. Component Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                TASK DECISION SERVICE                                 │
│                                                                                      │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                              API LAYER                                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │  │
│  │  │ REST API    │  │ WebSocket   │  │ Webhook     │  │ Message Queue       │   │  │
│  │  │ /api/v1/*   │  │ (real-time) │  │ Receivers   │  │ Consumer            │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                           CORE SERVICES                                        │  │
│  │                                                                                │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    InputNormalizerService                                 │ │  │
│  │  │  • TrelloAdapter    • TeamsAdapter    • EmailAdapter    • ManualAdapter  │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                        │                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    ClassificationService                                  │ │  │
│  │  │  • TaskClassifier                                                        │ │  │
│  │  │  • RoleAnalyzer                                                          │ │  │
│  │  │  • DeveloperAlternativeHandler                                           │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                        │                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    DecisionEngineService                                  │ │  │
│  │  │  • OpenAIClient          • FunctionCallHandler                           │ │  │
│  │  │  • RelationshipAnalyzer  • ScopeComparator                               │ │  │
│  │  │  • SubtaskMigrator       • ConflictDetector                              │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                        │                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    EstimationService                                      │ │  │
│  │  │  • TimeEstimator         • ComplexityAnalyzer                            │ │  │
│  │  │  • DueDateAnalyzer       • ConflictChecker                               │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                        │                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    AssignmentService                                      │ │  │
│  │  │  • UserScorer            • WorkloadCalculator                            │ │  │
│  │  │  • SkillMatcher          • AvailabilityChecker                           │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                        │                                       │  │
│  │  ┌──────────────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    ActionDispatcherService                                │ │  │
│  │  │  • CreateHandler         • UpdateHandler        • CommentHandler         │ │  │
│  │  │  • AssignHandler         • NotifyHandler        • ArchiveHandler         │ │  │
│  │  └──────────────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                                │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                        │                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────┐  │
│  │                           DATA LAYER                                           │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐   │  │
│  │  │ TaskRepo    │  │ UserRepo    │  │ AuditRepo   │  │ CacheService        │   │  │
│  │  │             │  │             │  │             │  │ (Redis)             │   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────────────┘   │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DETAILED DATA FLOW                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

    ┌──────────┐
    │  INPUT   │  Raw message from any source
    └────┬─────┘
         │
         ▼
┌─────────────────┐
│ 1. NORMALIZE    │  Convert to standard InputSchema
│    INPUT        │  Enrich sender info
└────────┬────────┘
         │
         ▼
         ┌───────────────────────────────────────┐
         │ InputSchema                           │
         │ {                                     │
         │   source_id, source_priority,         │
         │   task_content: { raw_text, ... },    │
         │   sender: { role, title, ... },       │
         │   context: { channel_type, ... }      │
         │ }                                     │
         └───────────────────┬───────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. CLASSIFICATION PHASE                                         │
│                                                                  │
│   ┌─────────────────┐     ┌─────────────────┐                   │
│   │ Analyze Content │────▶│ Analyze Role    │                   │
│   │ (task signals)  │     │ (sender weight) │                   │
│   └─────────────────┘     └────────┬────────┘                   │
│                                    │                             │
│                                    ▼                             │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ Classification Result                                    │   │
│   │ {                                                        │   │
│   │   type: VALID_TASK | IGNORE | ALT_ACTION,               │   │
│   │   adjusted_probability: 0.75,                            │   │
│   │   role_analysis: { category: EXECUTIVE, ... }            │   │
│   │ }                                                        │   │
│   └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ IGNORE   │      │ ALT_ACT  │      │ PROCEED  │
    │ (exit)   │      │ (comment │      │ (full    │
    │          │      │  update) │      │ algorithm│
    └──────────┘      └────┬─────┘      └────┬─────┘
                           │                 │
                           │                 ▼
                           │    ┌────────────────────────────────────────┐
                           │    │ 3. OPENAI DECISION ENGINE              │
                           │    │                                        │
                           │    │   ┌────────────────────────────────┐   │
                           │    │   │ System Prompt + Input          │   │
                           │    │   └───────────────┬────────────────┘   │
                           │    │                   │                    │
                           │    │                   ▼                    │
                           │    │   ┌────────────────────────────────┐   │
                           │    │   │ Function Calls (parallel)      │   │
                           │    │   │ • get_existing_tasks           │   │
                           │    │   │ • get_users_by_source          │   │
                           │    │   └───────────────┬────────────────┘   │
                           │    │                   │                    │
                           │    │                   ▼                    │
                           │    │   ┌────────────────────────────────┐   │
                           │    │   │ AI Analysis                    │   │
                           │    │   │ • Relationship (parent/sub)    │   │
                           │    │   │ • Time estimation              │   │
                           │    │   │ • Due date analysis            │   │
                           │    │   │ • User assignment              │   │
                           │    │   └───────────────┬────────────────┘   │
                           │    │                   │                    │
                           │    └───────────────────┼────────────────────┘
                           │                        │
                           └────────────────────────┤
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ 4. DECISION OUTPUT                                                                   │
│                                                                                      │
│   ┌─────────────────────────────────────────────────────────────────────────────┐   │
│   │ DecisionOutput                                                               │   │
│   │ {                                                                            │   │
│   │   classification: { type, probability, ... },                               │   │
│   │   role_analysis: { category, multiplier, ... },                             │   │
│   │   decision: { action: CREATE_NEW_TASK, confidence: 0.9 },                   │   │
│   │   task: { title, description, timing, assignment, ... },                    │   │
│   │   relationships: { parent_id, migrated_subtasks, ... },                     │   │
│   │   actions: [ { type: CREATE, payload: {...} }, ... ]                        │   │
│   │ }                                                                            │   │
│   └─────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ 5. ACTION DISPATCHER                                                                 │
│                                                                                      │
│   For each action in actions[]:                                                     │
│                                                                                      │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│   │   CREATE    │  │   UPDATE    │  │  ADD_COMMENT│  │   NOTIFY    │               │
│   │     ↓       │  │      ↓      │  │      ↓      │  │      ↓      │               │
│   │  TaskDB     │  │   TaskDB    │  │   TaskDB    │  │ Slack/Teams │               │
│   │  Trello     │  │   Trello    │  │   Trello    │  │   Email     │               │
│   │  Jira       │  │   Jira      │  │   Jira      │  │             │               │
│   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
                                            ┌──────────────┐
                                            │  AUDIT LOG   │
                                            │  (History)   │
                                            └──────────────┘
```

---

## 4. OpenAI Integration Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           OPENAI INTEGRATION DETAIL                                  │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              OpenAI Client Wrapper                                   │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Configuration                                        │    │
│  │  • Model: gpt-4o                                                            │    │
│  │  • Temperature: 0 (deterministic)                                           │    │
│  │  • Response Format: JSON Schema (structured output)                         │    │
│  │  • Tool Choice: auto                                                        │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Message Flow                                         │    │
│  │                                                                              │    │
│  │  1. System Prompt (algorithm rules, decision matrix)                        │    │
│  │                           │                                                  │    │
│  │                           ▼                                                  │    │
│  │  2. User Message (normalized input JSON)                                    │    │
│  │                           │                                                  │    │
│  │                           ▼                                                  │    │
│  │  3. Assistant Response (may include tool calls)                             │    │
│  │                           │                                                  │    │
│  │           ┌───────────────┴───────────────┐                                 │    │
│  │           ▼                               ▼                                 │    │
│  │  ┌─────────────────┐            ┌─────────────────┐                        │    │
│  │  │  Tool Call:     │            │  Tool Call:     │   (Parallel)           │    │
│  │  │  get_existing   │            │  get_users_by   │                        │    │
│  │  │  _tasks         │            │  _source        │                        │    │
│  │  └────────┬────────┘            └────────┬────────┘                        │    │
│  │           │                              │                                  │    │
│  │           ▼                              ▼                                  │    │
│  │  ┌─────────────────┐            ┌─────────────────┐                        │    │
│  │  │ Function Handler│            │ Function Handler│                        │    │
│  │  │ (DB Query)      │            │ (User Service)  │                        │    │
│  │  └────────┬────────┘            └────────┬────────┘                        │    │
│  │           │                              │                                  │    │
│  │           └──────────────┬───────────────┘                                 │    │
│  │                          ▼                                                  │    │
│  │  4. Tool Results (existing tasks, users)                                   │    │
│  │                          │                                                  │    │
│  │                          ▼                                                  │    │
│  │  5. Final Assistant Response (structured decision JSON)                    │    │
│  │                                                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                         Function Definitions                                 │    │
│  │                                                                              │    │
│  │  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐  │    │
│  │  │ get_existing_tasks  │  │ get_users_by_source │  │ get_task_subtasks   │  │    │
│  │  │ ─────────────────── │  │ ─────────────────── │  │ ─────────────────── │  │    │
│  │  │ • keywords[]        │  │ • source_id         │  │ • task_id           │  │    │
│  │  │ • project_id        │  │ • include_workload  │  │ • include_completed │  │    │
│  │  │ • status_filter[]   │  │ • skills_filter[]   │  │                     │  │    │
│  │  │ • due_date_range    │  │ • available_only    │  │                     │  │    │
│  │  │ • limit             │  │                     │  │                     │  │    │
│  │  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘  │    │
│  │                                                                              │    │
│  │  ┌─────────────────────┐                                                    │    │
│  │  │ get_similar_        │                                                    │    │
│  │  │ completed_tasks     │                                                    │    │
│  │  │ ─────────────────── │                                                    │    │
│  │  │ • keywords[]        │                                                    │    │
│  │  │ • task_type         │                                                    │    │
│  │  │ • limit             │                                                    │    │
│  │  └─────────────────────┘                                                    │    │
│  │                                                                              │    │
│  └─────────────────────────────────────────────────────────────────────────────┘    │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Database Schema

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              DATABASE ENTITIES                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────┐       ┌─────────────────────────┐
│         tasks           │       │         users           │
├─────────────────────────┤       ├─────────────────────────┤
│ id              UUID PK │       │ id              UUID PK │
│ title           VARCHAR │       │ name            VARCHAR │
│ description     TEXT    │       │ email           VARCHAR │
│ status          ENUM    │       │ role_category   ENUM    │
│ priority        ENUM    │◄──────│ role_title      VARCHAR │
│ type            ENUM    │       │ department      VARCHAR │
│ parent_id       UUID FK │───┐   │ skills          JSONB   │
│ assignee_id     UUID FK │───┼──▶│ workload_data   JSONB   │
│ reporter_id     UUID FK │───┤   │ availability    JSONB   │
│ project_id      UUID FK │   │   │ source_mappings JSONB   │
│ sprint_id       UUID FK │   │   │ created_at      TIMESTAMP│
│ due_date        TIMESTAMP   │   │ updated_at      TIMESTAMP│
│ estimated_min   INTEGER │   │   └─────────────────────────┘
│ actual_min      INTEGER │   │
│ labels          JSONB   │   │   ┌─────────────────────────┐
│ source          JSONB   │   │   │      task_comments      │
│ metadata        JSONB   │   │   ├─────────────────────────┤
│ created_at      TIMESTAMP   │   │ id              UUID PK │
│ updated_at      TIMESTAMP   │   │ task_id         UUID FK │──▶ tasks.id
│ started_at      TIMESTAMP   │   │ author_id       UUID FK │──▶ users.id
│ completed_at    TIMESTAMP   │   │ content         TEXT    │
└─────────────────────────┘   │   │ type            ENUM    │
         │                    │   │ source_message  JSONB   │
         │ (self-reference)   │   │ created_at      TIMESTAMP│
         └────────────────────┘   └─────────────────────────┘

┌─────────────────────────┐       ┌─────────────────────────┐
│    task_relationships   │       │       audit_log         │
├─────────────────────────┤       ├─────────────────────────┤
│ id              UUID PK │       │ id              UUID PK │
│ task_id         UUID FK │──▶    │ entity_type     VARCHAR │
│ related_task_id UUID FK │──▶    │ entity_id       UUID    │
│ relationship    ENUM    │       │ action          VARCHAR │
│  (blocks,blocked_by,    │       │ old_value       JSONB   │
│   related_to)           │       │ new_value       JSONB   │
│ created_at      TIMESTAMP       │ actor_id        UUID FK │
└─────────────────────────┘       │ actor_type      ENUM    │
                                  │  (user, system, ai)     │
┌─────────────────────────┐       │ decision_output JSONB   │
│    decision_history     │       │ created_at      TIMESTAMP│
├─────────────────────────┤       └─────────────────────────┘
│ id              UUID PK │
│ input_hash      VARCHAR │       ┌─────────────────────────┐
│ input_data      JSONB   │       │        projects         │
│ decision_output JSONB   │       ├─────────────────────────┤
│ model_used      VARCHAR │       │ id              UUID PK │
│ algorithm_ver   VARCHAR │       │ name            VARCHAR │
│ processing_ms   INTEGER │       │ key             VARCHAR │
│ created_at      TIMESTAMP       │ source_mappings JSONB   │
└─────────────────────────┘       │ settings        JSONB   │
                                  └─────────────────────────┘
```

---

## 6. Integration Points

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL INTEGRATIONS                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────┐    ┌──────────────────────────────────────┐
│            INPUT INTEGRATIONS            │    │           OUTPUT INTEGRATIONS        │
├──────────────────────────────────────────┤    ├──────────────────────────────────────┤
│                                          │    │                                      │
│  ┌────────────────────────────────────┐  │    │  ┌────────────────────────────────┐  │
│  │           Trello                   │  │    │  │           Trello               │  │
│  │  • Webhook: card.created          │  │    │  │  • POST /cards (create)        │  │
│  │  • Webhook: card.updated          │  │    │  │  • PUT /cards/{id} (update)    │  │
│  │  • GET /boards/{id}/cards         │  │    │  │  • POST /cards/{id}/comments   │  │
│  │  • GET /members                   │  │    │  │                                │  │
│  └────────────────────────────────────┘  │    │  └────────────────────────────────┘  │
│                                          │    │                                      │
│  ┌────────────────────────────────────┐  │    │  ┌────────────────────────────────┐  │
│  │        Microsoft Teams             │  │    │  │      Microsoft Teams           │  │
│  │  • Bot Framework Messages          │  │    │  │  • Adaptive Cards (notify)     │  │
│  │  • Graph API: /messages            │  │    │  │  • Graph API: /messages        │  │
│  │  • Graph API: /users               │  │    │  │                                │  │
│  │  • Webhook: channel messages       │  │    │  │                                │  │
│  └────────────────────────────────────┘  │    │  └────────────────────────────────┘  │
│                                          │    │                                      │
│  ┌────────────────────────────────────┐  │    │  ┌────────────────────────────────┐  │
│  │            Email                   │  │    │  │            Jira                │  │
│  │  • IMAP/POP3 (poll)               │  │    │  │  • POST /issue (create)        │  │
│  │  • Webhook (SendGrid, etc.)       │  │    │  │  • PUT /issue/{id} (update)    │  │
│  │  • Parse sender, subject, body    │  │    │  │  • POST /issue/{id}/comment    │  │
│  └────────────────────────────────────┘  │    │  └────────────────────────────────┘  │
│                                          │    │                                      │
│  ┌────────────────────────────────────┐  │    │  ┌────────────────────────────────┐  │
│  │           Slack                    │  │    │  │           Slack                │  │
│  │  • Events API: message             │  │    │  │  • chat.postMessage            │  │
│  │  • Slash commands                  │  │    │  │  • Blocks (rich messages)      │  │
│  │  • users.list                     │  │    │  │                                │  │
│  └────────────────────────────────────┘  │    │  └────────────────────────────────┘  │
│                                          │    │                                      │
└──────────────────────────────────────────┘    └──────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           OPENAI INTEGRATION                                         │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  Endpoint: https://api.openai.com/v1/chat/completions                               │
│  Model: gpt-4o (or gpt-4-turbo)                                                     │
│  Auth: Bearer token (API key)                                                       │
│                                                                                      │
│  Features Used:                                                                     │
│  • Function calling (tools)                                                         │
│  • Structured outputs (JSON schema response_format)                                 │
│  • Parallel function calls                                                          │
│                                                                                      │
│  Rate Limits Consideration:                                                         │
│  • Implement retry with exponential backoff                                         │
│  • Queue requests during high load                                                  │
│  • Cache similar decisions (by input hash)                                          │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           DEPLOYMENT (AWS EXAMPLE)                                   │
└─────────────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   CloudFront    │
                              │   (CDN/Edge)    │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │   API Gateway   │
                              │   (REST/WS)     │
                              └────────┬────────┘
                                       │
                 ┌─────────────────────┼─────────────────────┐
                 │                     │                     │
        ┌────────▼────────┐   ┌────────▼────────┐   ┌────────▼────────┐
        │  Lambda/ECS     │   │  Lambda/ECS     │   │  Lambda/ECS     │
        │  (API Service)  │   │  (Webhook Rx)   │   │  (Queue Worker) │
        └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
                 │                     │                     │
                 └─────────────────────┼─────────────────────┘
                                       │
                              ┌────────▼────────┐
                              │   SQS Queue     │
                              │ (async process) │
                              └────────┬────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
┌───────▼───────┐            ┌─────────▼─────────┐          ┌────────▼────────┐
│   RDS/Aurora  │            │   ElastiCache     │          │   OpenAI API    │
│  (PostgreSQL) │            │   (Redis)         │          │   (External)    │
│               │            │                   │          │                 │
│  • Tasks      │            │  • Session cache  │          │  • GPT-4o       │
│  • Users      │            │  • Rate limiting  │          │  • Functions    │
│  • Audit      │            │  • Decision cache │          │                 │
└───────────────┘            └───────────────────┘          └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           MONITORING & OBSERVABILITY                                 │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │
│  │ CloudWatch  │  │ X-Ray       │  │ OpenSearch  │  │ SNS/PagerDuty│               │
│  │ (Metrics)   │  │ (Tracing)   │  │ (Logs)      │  │ (Alerts)    │                │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘                │
│                                                                                      │
│  Key Metrics:                                                                       │
│  • Decisions per minute                                                             │
│  • Classification distribution                                                      │
│  • OpenAI latency (p50, p95, p99)                                                  │
│  • Error rate by type                                                               │
│  • Review queue size                                                                │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. Security Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              SECURITY LAYERS                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: NETWORK                                                                     │
│  • VPC isolation                                                                    │
│  • Private subnets for database/cache                                               │
│  • Security groups (least privilege)                                                │
│  • WAF on API Gateway                                                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 2: AUTHENTICATION                                                              │
│  • API keys for service-to-service                                                  │
│  • OAuth 2.0 / JWT for user requests                                                │
│  • Webhook signature verification                                                   │
│  • Bot token validation (Teams, Slack)                                              │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 3: AUTHORIZATION                                                               │
│  • Role-based access control (RBAC)                                                 │
│  • Source-level permissions                                                         │
│  • Project-level isolation                                                          │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 4: DATA                                                                        │
│  • Encryption at rest (RDS, S3)                                                     │
│  • Encryption in transit (TLS 1.3)                                                  │
│  • PII handling (masking in logs)                                                   │
│  • OpenAI data: no training on API data                                             │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│ LAYER 5: SECRETS                                                                     │
│  • AWS Secrets Manager / HashiCorp Vault                                            │
│  • OpenAI API key rotation                                                          │
│  • Integration tokens (Trello, Teams, etc.)                                         │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 9. Technology Stack (Recommended)

| Layer | Technology | Rationale |
|-------|------------|-----------|
| **Language** | Python 3.11+ / Node.js 20+ | Best OpenAI SDK support |
| **Framework** | FastAPI / NestJS | Async, typed, OpenAPI docs |
| **Database** | PostgreSQL 15 | JSONB support, reliability |
| **Cache** | Redis 7 | Rate limiting, session cache |
| **Queue** | AWS SQS / RabbitMQ | Async processing, retries |
| **AI** | OpenAI GPT-4o | Best reasoning, function calling |
| **Hosting** | AWS ECS/Lambda | Scalable, serverless options |
| **Monitoring** | DataDog / CloudWatch | Comprehensive observability |

---

## 10. Key Design Decisions

| Decision | Choice | Reason |
|----------|--------|--------|
| **Sync vs Async** | Async (queue-based) | Handle rate limits, retries gracefully |
| **AI Model** | GPT-4o | Best accuracy for complex decisions |
| **Temperature** | 0 | Deterministic, reproducible decisions |
| **Caching** | Cache by input hash | Reduce costs, improve latency |
| **Audit** | Full decision logging | Debugging, compliance, learning |
| **Multi-tenancy** | Project-level isolation | Support multiple teams |
