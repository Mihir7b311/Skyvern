# 4.1 Main Agent Architecture
## `skyvern/forge/sdk/agent.py` 🔥 **CRITICAL**

---

### 🎯 **Purpose & Role**
The **ForgeAgent** is the central orchestrator of Skyvern's automation engine, managing:
- Task execution flow
- Step management  
- Browser action coordination
- AI integration

---

### 🏗️ **Core Architecture**

```mermaid
classDiagram
    class ForgeAgent {
        +AsyncOperationPool async_operation_pool
        +__init__()
        +create_task_and_step_from_block()
        +execute_step()
        +_handle_agent_errors()
        +_get_action_plan()
        +_execute_action()
    }
    
    class AsyncOperationPool {
        +execute_async_operation()
        +cleanup_operations()
    }
    
    class ActionLinkedNode {
        +Action action
        +ActionLinkedNode next
    }
    
    ForgeAgent --> AsyncOperationPool
    ForgeAgent --> ActionLinkedNode
    ForgeAgent --> "multiple" Action
```

---

### 🔄 **Task Execution Flow**

```mermaid
sequenceDiagram
    participant Client
    participant Agent as ForgeAgent
    participant Context as SkyvernContext
    participant Browser as BrowserManager
    participant AI as LLM API
    
    Client->>Agent: execute_task()
    Agent->>Context: set_context()
    Agent->>Browser: get_page()
    
    loop For each step
        Agent->>Browser: scrape_website()
        Agent->>AI: get_action_plan()
        AI-->>Agent: actions_list
        
        loop For each action
            Agent->>Browser: execute_action()
            Browser-->>Agent: action_result
        end
        
        Agent->>Agent: validate_step()
        Agent->>Context: update_context()
    end
    
    Agent-->>Client: execution_result
```

---

### ⚙️ **Key Methods Breakdown**

#### **🚀 execute_step()**
```python
async def execute_step(
    self,
    organization: Organization,
    task: Task,
    step: Step,
    api_key: str | None = None,
    engine: RunEngine = RunEngine.skyvern_v1,
) -> tuple[Step, list[ActionResult], ScrapedPage]:
```

**Purpose**: Main step execution orchestrator
- Scrapes the current page
- Gets action plan from AI
- Executes actions sequentially
- Handles errors and retries

---

#### **🎯 _get_action_plan()**
```python
async def _get_action_plan(
    self,
    scraped_page: ScrapedPage,
    task: Task,
    step: Step,
) -> list[Action]:
```

**Purpose**: AI-powered action planning
- Analyzes scraped page content
- Sends context to LLM
- Parses AI response into actions
- Validates action feasibility

---

#### **⚡ _execute_action()**
```python
async def _execute_action(
    self,
    action: Action,
    page: Page,
    scraped_page: ScrapedPage,
    step: Step,
    task: Task,
) -> ActionResult:
```

**Purpose**: Individual action execution
- Handles browser interactions
- Manages action-specific logic
- Returns execution results
- Updates page state

---

### 🧠 **AI Integration Pattern**

```mermaid
graph LR
    subgraph "AI Action Planning"
        A[Scraped Page] --> B[Prompt Engineering]
        B --> C[LLM API Call]
        C --> D[Response Parsing]
        D --> E[Action Validation]
        E --> F[Action List]
    end
    
    subgraph "Action Execution"
        F --> G[Action Handler]
        G --> H[Browser Interaction]
        H --> I[Result Validation]
        I --> J[State Update]
    end
```

---

### 🔧 **Error Handling Strategy**

```mermaid
flowchart TD
    A[Action Execution] --> B{Success?}
    B -->|Yes| C[Continue Next Action]
    B -->|No| D[Error Analysis]
    
    D --> E{Retryable?}
    E -->|Yes| F[Increment Retry Count]
    F --> G{Max Retries?}
    G -->|No| A
    G -->|Yes| H[Mark Step Failed]
    
    E -->|No| I[Critical Error]
    I --> J[Terminate Task]
    
    H --> K[Continue Next Step]
```

---

### 🎛️ **Configuration Integration**

The agent integrates with multiple configuration sources:

| Component | Purpose | Example |
|-----------|---------|---------|
| `settings_manager` | Global config | Browser timeouts, AI model selection |
| `skyvern_context` | Request state | Current task, organization, session |
| `task.parameters` | Task-specific | Custom prompts, extraction schemas |

---

### 🔍 **Key Performance Features**

#### **Async Operation Pool**
- Parallel execution of non-blocking operations
- Resource management and cleanup
- Background task handling

#### **Action Caching**
- Reuses previously computed action plans
- Reduces AI API calls
- Improves response times

#### **Smart Retry Logic**
- Differentiates between retryable and fatal errors
- Exponential backoff for network issues
- Context preservation across retries

---

### 🎯 **Next: Context Management**
Understanding how global state is managed across execution...