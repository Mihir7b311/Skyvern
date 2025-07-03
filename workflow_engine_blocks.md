# 🧩 Block Models Deep Dive
## Workflow Engine - Block System Analysis

---

## Slide 4: Block Type Definitions {#slide-4}

### Complete Block Type Enumeration

```python
class BlockType(StrEnum):
    TASK = "task"                    # Browser automation tasks
    TaskV2 = "task_v2"              # Enhanced task execution
    FOR_LOOP = "for_loop"           # Iteration control
    CODE = "code"                   # Custom Python execution
    TEXT_PROMPT = "text_prompt"     # LLM text generation
    DOWNLOAD_TO_S3 = "download_to_s3"     # File downloads
    UPLOAD_TO_S3 = "upload_to_s3"         # File uploads
    FILE_UPLOAD = "file_upload"           # Local file handling
    SEND_EMAIL = "send_email"             # Email notifications
    FILE_URL_PARSER = "file_url_parser"   # URL content parsing
    VALIDATION = "validation"             # Data validation
    ACTION = "action"                     # Single browser actions
    NAVIGATION = "navigation"             # Page navigation
    EXTRACTION = "extraction"             # Data extraction
    LOGIN = "login"                       # Authentication
    WAIT = "wait"                         # Timing control
    FILE_DOWNLOAD = "file_download"       # File retrieval
    GOTO_URL = "goto_url"                # URL navigation
    PDF_PARSER = "pdf_parser"            # PDF processing
    HTTP_REQUEST = "http_request"        # API calls
```

### Block Categories:

```mermaid
mindmap
  root((Block Types))
    Browser Automation
      TASK
      TaskV2
      ACTION
      NAVIGATION
      EXTRACTION
      LOGIN
    Control Flow
      FOR_LOOP
      VALIDATION
      WAIT
    Data Processing
      CODE
      TEXT_PROMPT
      PDF_PARSER
      FILE_URL_PARSER
    File Operations
      FILE_UPLOAD
      FILE_DOWNLOAD
      DOWNLOAD_TO_S3
      UPLOAD_TO_S3
    Communication
      SEND_EMAIL
      HTTP_REQUEST
    Navigation
      GOTO_URL
```

---

## Slide 5: Block Execution Patterns {#slide-5}

### Block Execution Flow

```mermaid
sequenceDiagram
    participant WE as Workflow Engine
    participant B as Block
    participant CM as Context Manager
    participant EE as Execution Engine
    participant DB as Database
    
    WE->>B: execute(workflow_run_id, organization_id)
    B->>CM: get_workflow_run_context()
    CM-->>B: WorkflowRunContext
    
    B->>B: format_parameters_from_context()
    B->>EE: execute_block_logic()
    
    alt Success
        EE-->>B: execution_result
        B->>CM: record_output_parameter_value()
        B->>DB: update_workflow_run_block(status="completed")
        B-->>WE: BlockResult(success=True)
    else Failure
        EE-->>B: exception/error
        B->>DB: update_workflow_run_block(status="failed")
        B-->>WE: BlockResult(success=False, failure_reason)
    end
```

### Parameter Resolution System:

```mermaid
graph LR
    subgraph "Parameter Types"
        A[WorkflowParameter]
        B[ContextParameter]
        C[OutputParameter]
        D[AWSSecretParameter]
    end
    
    subgraph "Resolution Process"
        E[Template String]
        F[Jinja2 Sandbox]
        G[Context Values]
        H[Resolved Value]
    end
    
    A --> E
    B --> E
    C --> G
    D --> G
    
    E --> F
    G --> F
    F --> H
```

---

## Slide 6: Context Management System {#slide-6}

### WorkflowRunContext Architecture

```mermaid
classDiagram
    class WorkflowRunContext {
        +workflow_run_id: str
        +values: dict
        +secrets: dict
        +metadata: dict
        +parameter_registry: dict
        +has_value(key: str) bool
        +get_value(key: str) Any
        +register_parameter(param) void
        +register_output_parameter_value() void
        +get_block_metadata(label: str) dict
    }
    
    class BlockMetadata {
        +current_index: int
        +current_item: Any
        +current_value: Any
        +parent_block_id: str
    }
    
    class WorkflowContextManager {
        +workflow_run_contexts: dict
        +aws_client: AsyncAWSClient
        +create_workflow_run_context() WorkflowRunContext
        +get_workflow_run_context() WorkflowRunContext
        +cleanup_workflow_run_context() void
    }
    
    WorkflowRunContext --> BlockMetadata
    WorkflowContextManager --> WorkflowRunContext
```

### Context Lifecycle:

```mermaid
stateDiagram-v2
    [*] --> Initialize: create_workflow_run_context()
    Initialize --> Active: register_parameters()
    Active --> Active: register_output_values()
    Active --> Active: update_block_metadata()
    Active --> Cleanup: workflow_complete
    Cleanup --> [*]: cleanup_workflow_run_context()
    
    Active --> Error: exception_thrown
    Error --> Cleanup: cleanup_required
```

---

## Slide 7: Parameter System Deep Dive {#slide-7}

### Parameter Type Hierarchy

```mermaid
classDiagram
    class Parameter {
        <<abstract>>
        +key: str
        +description: str
    }
    
    class WorkflowParameter {
        +workflow_parameter_type: WorkflowParameterType
        +default_value: Any
        +value: Any
    }
    
    class ContextParameter {
        +source_parameter_key: str
        +source: Parameter
    }
    
    class OutputParameter {
        +output_parameter_id: str
        +workflow_id: str
    }
    
    class AWSSecretParameter {
        +aws_secret_name: str
        +aws_secret_key: str
    }
    
    Parameter <|-- WorkflowParameter
    Parameter <|-- ContextParameter
    Parameter <|-- OutputParameter
    Parameter <|-- AWSSecretParameter
```

### Parameter Resolution Flow:

```mermaid
flowchart TD
    A[Block Parameter Request] --> B{Parameter Type?}
    
    B -->|WorkflowParameter| C[Get Default/Set Value]
    B -->|ContextParameter| D[Resolve Source Parameter]
    B -->|OutputParameter| E[Get from Previous Block Output]
    B -->|AWSSecretParameter| F[Fetch from AWS Secrets Manager]
    
    C --> G[Jinja2 Template Processing]
    D --> G
    E --> G
    F --> G
    
    G --> H[Inject Context Variables]
    H --> I[Return Resolved Value]
    
    subgraph "Context Variables"
        J[current_index]
        K[current_item]
        L[current_value]
        M[block_metadata]
    end
    
    H --> J
    H --> K
    H --> L
    H --> M
```

---

## Continue to [Execution Engine →](workflow_engine_execution.md)

---

*This section details the block models and parameter system that form the foundation of Skyvern's workflow engine.*