# ⚙️ Execution Engine & Orchestration
## Workflow Engine - Service Layer Analysis

---

## Slide 8: Workflow Service Architecture {#slide-8}

### WorkflowService Core Components

```mermaid
classDiagram
    class WorkflowService {
        +create_workflow_from_request() Workflow
        +setup_workflow_run() WorkflowRun
        +execute_workflow() void
        +mark_workflow_run_as_running() WorkflowRun
        +mark_workflow_run_as_completed() WorkflowRun
        +mark_workflow_run_as_failed() WorkflowRun
        +build_workflow_run_status_response() WorkflowRunResponse
        +get_workflow_runs() list[WorkflowRun]
    }
    
    class WorkflowDefinition {
        +parameters: list[Parameter]
        +blocks: list[Block]
    }
    
    class WorkflowRun {
        +workflow_run_id: str
        +status: WorkflowRunStatus
        +start_time: datetime
        +end_time: datetime
        +failure_reason: str
    }
    
    class WorkflowRunBlock {
        +workflow_run_block_id: str
        +block_type: BlockType
        +status: BlockStatus
        +output: Any
        +failure_reason: str
    }
    
    WorkflowService --> WorkflowDefinition
    WorkflowService --> WorkflowRun
    WorkflowRun --> WorkflowRunBlock
```

### Workflow Execution States:

```mermaid
stateDiagram-v2
    [*] --> created: setup_workflow_run()
    created --> queued: submit_to_executor
    queued --> running: start_execution
    running --> running: executing_blocks
    running --> completed: all_blocks_success
    running --> failed: block_failed_critical
    running --> terminated: manual_termination
    running --> canceled: user_cancellation
    running --> timed_out: execution_timeout
    
    completed --> [*]
    failed --> [*]
    terminated --> [*]
    canceled --> [*]
    timed_out --> [*]
```

---

## Slide 9: Block Execution Orchestration {#slide-9}

### Block Execution Sequence

```mermaid
sequenceDiagram
    participant API as API Endpoint
    participant WS as WorkflowService
    participant EE as ExecutionEngine
    participant WCM as WorkflowContextManager
    participant BE as BlockExecutor
    participant DB as Database
    
    API->>WS: execute_workflow_request()
    WS->>DB: create_workflow_run()
    WS->>WCM: create_workflow_run_context()
    WS->>EE: AsyncExecutorFactory.execute_workflow()
    
    EE->>DB: get_workflow_definition()
    EE->>EE: build_execution_plan()
    
    loop For Each Block
        EE->>BE: execute_block()
        BE->>WCM: get_workflow_run_context()
        BE->>BE: format_parameters()
        BE->>BE: execute_block_logic()
        
        alt Block Success
            BE->>WCM: register_output_parameter()
            BE->>DB: update_block_status(completed)
        else Block Failure
            BE->>DB: update_block_status(failed)
            alt continue_on_failure = false
                BE-->>EE: terminate_workflow
            end
        end
        
        BE-->>EE: BlockResult
    end
    
    EE->>WS: mark_workflow_as_completed()
    WS-->>API: WorkflowRunResponse
```

### Parallel Block Execution:

```mermaid
graph TB
    subgraph "Workflow Execution"
        A[Start] --> B{Fork Point?}
        B -->|No| C[Sequential Block]
        B -->|Yes| D[Parallel Branch 1]
        B -->|Yes| E[Parallel Branch 2]
        B -->|Yes| F[Parallel Branch 3]
        
        C --> G[Next Block]
        D --> H[Join Point]
        E --> H
        F --> H
        H --> I[Continue Workflow]
        
        G --> J[End]
        I --> J
    end
```

---

## Slide 10: Data Flow & State Management {#slide-10}

### Data Flow Architecture

```mermaid
flowchart LR
    subgraph "Input Layer"
        A[Workflow Request]
        B[Parameters]
        C[Secrets]
    end
    
    subgraph "Processing Layer"
        D[Context Manager]
        E[Parameter Registry]
        F[Block Executor]
    end
    
    subgraph "State Layer"
        G[Workflow Run Context]
        H[Block Metadata]
        I[Output Parameters]
    end
    
    subgraph "Output Layer"
        J[Block Results]
        K[Workflow Response]
        L[Artifacts]
    end
    
    A --> D
    B --> E
    C --> E
    
    D --> G
    E --> F
    F --> H
    
    G --> I
    H --> J
    I --> K
    J --> L
```

### State Persistence Strategy:

```mermaid
graph TB
    subgraph "Memory State"
        A[WorkflowRunContext]
        B[Block Metadata]
        C[Parameter Values]
    end
    
    subgraph "Database State"
        D[WorkflowRun Table]
        E[WorkflowRunBlock Table]
        F[WorkflowRunParameter Table]
    end
    
    subgraph "External State"
        G[S3 Artifacts]
        H[AWS Secrets]
        I[Browser Sessions]
    end
    
    A -.->|Sync| D
    B -.->|Sync| E
    C -.->|Sync| F
    
    A -->|Reference| G
    A -->|Fetch| H
    A -->|Manage| I
```

---

## Slide 11: Error Handling & Recovery {#slide-11}

### Exception Hierarchy

```mermaid
classDiagram
    class SkyvernException {
        <<abstract>>
    }
    
    class WorkflowDefinitionHasDuplicateParameterKeys {
        +duplicate_keys: list[str]
    }
    
    class OutputParameterKeyCollisionError {
        +colliding_key: str
        +existing_parameter: str
    }
    
    class CustomizedCodeException {
        +code: str
        +error_message: str
    }
    
    class InsecureCodeDetected {
        +detected_patterns: list[str]
    }
    
    class NoIterableValueFound {
        +parameter_key: str
    }
    
    SkyvernException <|-- WorkflowDefinitionHasDuplicateParameterKeys
    SkyvernException <|-- OutputParameterKeyCollisionError
    SkyvernException <|-- CustomizedCodeException
    SkyvernException <|-- InsecureCodeDetected
    SkyvernException <|-- NoIterableValueFound
```

### Error Recovery Patterns:

```mermaid
flowchart TD
    A[Block Execution] --> B{Exception Occurred?}
    B -->|No| C[Block Success]
    B -->|Yes| D{continue_on_failure?}
    
    D -->|True| E[Log Error & Continue]
    D -->|False| F{Retryable Error?}
    
    F -->|Yes| G{Retry Attempts < Max?}
    F -->|No| H[Terminate Workflow]
    
    G -->|Yes| I[Wait & Retry]
    G -->|No| H
    
    I --> A
    E --> J[Next Block]
    C --> J
    H --> K[Mark Workflow Failed]
    J --> L[Continue Execution]
```

---

## Slide 12: Performance & Optimization {#slide-12}

### Execution Optimization Strategies

```mermaid
mindmap
  root((Performance Optimization))
    Parallel Execution
      Block-level Parallelism
      Resource Pool Management
      Async Coordination
    Caching Strategy
      Action Result Caching
      Parameter Value Caching
      Browser Session Reuse
    Resource Management
      Memory Optimization
      Database Connection Pooling
      Browser Resource Cleanup
    Monitoring
      Execution Metrics
      Performance Profiling
      Error Rate Tracking
```

### Scalability Architecture:

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[API Gateway]
    end
    
    subgraph "Application Layer"
        A1[App Instance 1]
        A2[App Instance 2]
        A3[App Instance N]
    end
    
    subgraph "Execution Layer"
        E1[Executor Pool 1]
        E2[Executor Pool 2]
        E3[Executor Pool N]
    end
    
    subgraph "Storage Layer"
        DB[(Database Cluster)]
        S3[(S3 Storage)]
        REDIS[(Redis Cache)]
    end
    
    LB --> A1
    LB --> A2
    LB --> A3
    
    A1 --> E1
    A2 --> E2
    A3 --> E3
    
    E1 --> DB
    E2 --> DB
    E3 --> DB
    
    E1 --> S3
    E2 --> S3
    E3 --> S3
    
    A1 --> REDIS
    A2 --> REDIS
    A3 --> REDIS
```

---

## Continue to [Implementation Examples →](workflow_engine_examples.md)

---

*This section covers the execution engine, orchestration patterns, and performance optimization strategies of the Skyvern workflow system.*