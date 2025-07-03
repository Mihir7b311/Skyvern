# Phase 2: Database Layer
## Skyvern Repository Deep Dive

---

## 📋 Phase Overview

**Phase 2 Focus**: Comprehensive database architecture supporting tasks, workflows, organizations, and automation metadata.

**Timeline**: Week 1-2 of the 12-phase learning plan

**Prerequisites**: Phase 1 (Foundation & Configuration) completed

**Importance**: The database layer is the persistent foundation that stores all automation data, user information, and execution history.

---

## 🎯 Learning Goals for Phase 2

- ✅ **Data Model Mastery**: Understand all entities and their relationships
- ✅ **Query Patterns**: Know common database operations and optimizations
- ✅ **Schema Evolution**: Grasp migration and versioning strategies
- ✅ **Performance Optimization**: Understand indexing and query optimization
- ✅ **Data Validation**: Know Pydantic schema usage for data integrity

---

## 🗂️ Critical Files to Study (Priority Order)

### 🔥 **CRITICAL** Files (Must Master)
1. **`skyvern/forge/sdk/db/models.py`** - All database table definitions
2. **`skyvern/forge/sdk/db/client.py`** - Database connection and operations
3. **`skyvern/forge/sdk/schemas/tasks.py`** - Task-related data structures
4. **`skyvern/forge/sdk/db/enums.py`** - Database enumerations

### 🔶 **IMPORTANT** Files (Should Understand)
5. **`skyvern/forge/sdk/db/converters.py`** - Model to schema conversions
6. **`skyvern/forge/sdk/schemas/organizations.py`** - Organization schemas
7. **`skyvern/forge/sdk/schemas/workflow_runs.py`** - Workflow execution schemas
8. **`skyvern/forge/sdk/db/id_generators.py`** - ID generation logic

---

## 🗄️ Database Architecture Overview

### Technology Stack
| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| **Database** | PostgreSQL | 15+ | Primary data store |
| **ORM** | SQLAlchemy | 2.0+ | Object-relational mapping |
| **Migrations** | Alembic | Latest | Schema version control |
| **Connection Pool** | psycopg3 | Latest | Connection management |
| **Validation** | Pydantic | 2.0+ | Data validation |

### Database Connection Configuration
```python
# From skyvern/config.py
DATABASE_STRING: str = "postgresql+psycopg://skyvern@localhost/skyvern"
DATABASE_STATEMENT_TIMEOUT_MS: int = 60000
DISABLE_CONNECTION_POOL: bool = False

# Connection pool settings
POOL_SIZE = 20
MAX_OVERFLOW = 30
POOL_TIMEOUT = 30
POOL_RECYCLE = 3600
```

---

## 📁 File 1: `skyvern/forge/sdk/db/models.py` 🔥 **CRITICAL**

### Purpose
Defines all SQLAlchemy database models representing the core entities in Skyvern's data architecture.

### Core Entity Relationship Diagram

```mermaid
erDiagram
    Organization ||--o{ Task : owns
    Organization ||--o{ Workflow : owns
    Organization ||--o{ ApiKey : has
    Organization ||--o{ User : contains
    
    Task ||--o{ Step : contains
    Task ||--o{ Artifact : generates
    Task ||--o{ TaskLog : logs
    
    Workflow ||--o{ WorkflowRun : executes
    WorkflowRun ||--o{ WorkflowBlock : contains
    WorkflowRun ||--o{ WorkflowRunLog : logs
    
    Step ||--o{ Action : performs
    Step ||--o{ Screenshot : captures
    Step ||--o{ StepLog : logs
    
    User ||--o{ Task : creates
    User ||--o{ WorkflowRun : initiates
    
    BrowserSession ||--o{ Task : supports
    BrowserSession ||--o{ WorkflowRun : supports
```

### 1.1 Organization Model

```python
class Organization(Base):
    __tablename__ = "organizations"
    
    # Primary key
    organization_id = Column(String, primary_key=True, default=generate_organization_id)
    
    # Basic information
    organization_name = Column(String, nullable=False, unique=True)
    domain = Column(String, nullable=True)
    
    # Configuration
    webhook_callback_url = Column(String, nullable=True)
    max_steps_per_run = Column(Integer, default=10)
    max_retries_per_step = Column(Integer, default=3)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # Status and settings
    is_active = Column(Boolean, default=True)
    settings = Column(JSON, default=dict)
    
    # Relationships
    tasks = relationship("Task", back_populates="organization", cascade="all, delete-orphan")
    workflows = relationship("Workflow", back_populates="organization", cascade="all, delete-orphan")
    api_keys = relationship("ApiKey", back_populates="organization", cascade="all, delete-orphan")
    users = relationship("User", back_populates="organization", cascade="all, delete-orphan")
    
    # Indexes
    __table_args__ = (
        Index('ix_organizations_name', 'organization_name'),
        Index('ix_organizations_domain', 'domain'),
        Index('ix_organizations_active', 'is_active'),
    )
```

**Organization Schema Details:**
| Field | Type | Constraints | Purpose |
|-------|------|-------------|---------|
| `organization_id` | String | Primary Key, Auto-generated | Unique identifier |
| `organization_name` | String | Not Null, Unique | Display name |
| `domain` | String | Nullable | Associated domain |
| `webhook_callback_url` | String | Nullable | Callback for events |
| `max_steps_per_run` | Integer | Default: 10 | Execution limits |
| `is_active` | Boolean | Default: True | Status flag |
| `settings` | JSON | Default: {} | Custom configuration |

### 1.2 Task Model

```python
class Task(Base):
    __tablename__ = "tasks"
    
    # Primary key
    task_id = Column(String, primary_key=True, default=generate_task_id)
    
    # Foreign keys
    organization_id = Column(String, ForeignKey("organizations.organization_id"), nullable=False)
    created_by = Column(String, ForeignKey("users.user_id"), nullable=True)
    
    # Task configuration
    status = Column(Enum(TaskStatus), default=TaskStatus.created)
    url = Column(Text, nullable=False)
    navigation_goal = Column(Text, nullable=False)
    data_extraction_goal = Column(Text, nullable=True)
    navigation_payload = Column(JSON, nullable=True)
    
    # Execution metadata
    proxy_location = Column(String, nullable=True)
    webhook_callback_url = Column(String, nullable=True)
    totp_verification_url = Column(String, nullable=True)
    totp_identifier = Column(String, nullable=True)
    
    # Results and metrics
    extracted_information = Column(JSON, nullable=True)
    failure_reason = Column(Text, nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    modified_at = Column(DateTime(timezone=True), onupdate=func.now())
    completed_at = Column(DateTime(timezone=True), nullable=True)
    
    # Relationships
    organization = relationship("Organization", back_populates="tasks")
    creator = relationship("User", back_populates="created_tasks")
    steps = relationship("Step", back_populates="task", cascade="all, delete-orphan", order_by="Step.order")
    artifacts = relationship("Artifact", back_populates="task", cascade="all, delete-orphan")
    task_logs = relationship("TaskLog", back_populates="task", cascade="all, delete-orphan")
    
    # Indexes
    __table_args__ = (
        Index('ix_tasks_organization_id', 'organization_id'),
        Index('ix_tasks_status', 'status'),
        Index('ix_tasks_created_at', 'created_at'),
        Index('ix_tasks_created_by', 'created_by'),
        Index('ix_tasks_status_org', 'status', 'organization_id'),
    )
```

**Task Status Lifecycle:**
```mermaid
stateDiagram-v2
    [*] --> created
    created --> queued
    queued --> running
    running --> completed
    running --> failed
    running --> cancelled
    failed --> retrying
    retrying --> running
    retrying --> failed
    cancelled --> [*]
    completed --> [*]
    failed --> [*]
```

### 1.3 Step Model

```python
class Step(Base):
    __tablename__ = "steps"
    
    # Primary key
    step_id = Column(String, primary_key=True, default=generate_step_id)
    
    # Foreign keys
    task_id = Column(String, ForeignKey("tasks.task_id"), nullable=False)
    
    # Step metadata
    status = Column(Enum(StepStatus), default=StepStatus.created)
    order = Column(Integer, nullable=False)
    retry_index = Column(Integer, default=0)
    
    # Step content
    step_type = Column(String, nullable=False)  # "action", "validation", "extraction"
    input_data = Column(JSON, nullable=True)
    output_data = Column(JSON, nullable=True)
    
    # Execution details
    is_goal_achieved = Column(Boolean, nullable=True)
    output_parameter_key = Column(String, nullable=True)
    
    # Error handling
    failure_reason = Column(Text, nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    started_at = Column(DateTime(timezone=True), nullable=True)
    completed_at = Column(DateTime(timezone=True), nullable=True)
    
    # Relationships
    task = relationship("Task", back_populates="steps")
    actions = relationship("Action", back_populates="step", cascade="all, delete-orphan")
    screenshots = relationship("Screenshot", back_populates="step", cascade="all, delete-orphan")
    step_logs = relationship("StepLog", back_populates="step", cascade="all, delete-orphan")
    
    # Indexes
    __table_args__ = (
        Index('ix_steps_task_id', 'task_id'),
        Index('ix_steps_status', 'status'),
        Index('ix_steps_order', 'task_id', 'order'),
        Index('ix_steps_created_at', 'created_at'),
    )
```

### 1.4 Workflow Models

#### Workflow Template Model
```python
class Workflow(Base):
    __tablename__ = "workflows"
    
    # Primary key
    workflow_id = Column(String, primary_key=True, default=generate_workflow_id)
    workflow_permanent_id = Column(String, unique=True, nullable=False, default=generate_workflow_permanent_id)
    
    # Foreign keys
    organization_id = Column(String, ForeignKey("organizations.organization_id"), nullable=False)
    created_by = Column(String, ForeignKey("users.user_id"), nullable=True)
    
    # Workflow metadata
    title = Column(String, nullable=False)
    description = Column(Text, nullable=True)
    version = Column(Integer, default=1)
    is_active = Column(Boolean, default=True)
    
    # Workflow definition
    workflow_definition = Column(JSON, nullable=False)  # Blocks and connections
    parameter_schema = Column(JSON, nullable=True)      # Input parameter definitions
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    modified_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # Relationships
    organization = relationship("Organization", back_populates="workflows")
    creator = relationship("User", back_populates="created_workflows")
    workflow_runs = relationship("WorkflowRun", back_populates="workflow", cascade="all, delete-orphan")
    
    # Indexes
    __table_args__ = (
        Index('ix_workflows_organization_id', 'organization_id'),
        Index('ix_workflows_permanent_id', 'workflow_permanent_id'),
        Index('ix_workflows_active', 'is_active'),
        Index('ix_workflows_created_by', 'created_by'),
    )
```

#### Workflow Run Model
```python
class WorkflowRun(Base):
    __tablename__ = "workflow_runs"
    
    # Primary key
    workflow_run_id = Column(String, primary_key=True, default=generate_workflow_run_id)
    
    # Foreign keys
    workflow_id = Column(String, ForeignKey("workflows.workflow_id"), nullable=False)
    organization_id = Column(String, ForeignKey("organizations.organization_id"), nullable=False)
    initiated_by = Column(String, ForeignKey("users.user_id"), nullable=True)
    
    # Execution metadata
    status = Column(Enum(WorkflowRunStatus), default=WorkflowRunStatus.created)
    current_block_index = Column(Integer, default=0)
    
    # Input/Output
    parameters = Column(JSON, nullable=True)           # Input parameters
    outputs = Column(JSON, nullable=True)              # Final outputs
    
    # Error handling
    failure_reason = Column(Text, nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    started_at = Column(DateTime(timezone=True), nullable=True)
    completed_at = Column(DateTime(timezone=True), nullable=True)
    
    # Relationships
    workflow = relationship("Workflow", back_populates="workflow_runs")
    organization = relationship("Organization")
    initiator = relationship("User", back_populates="initiated_workflow_runs")
    workflow_blocks = relationship("WorkflowBlock", back_populates="workflow_run", cascade="all, delete-orphan")
    workflow_run_logs = relationship("WorkflowRunLog", back_populates="workflow_run", cascade="all, delete-orphan")
    
    # Indexes
    __table_args__ = (
        Index('ix_workflow_runs_workflow_id', 'workflow_id'),
        Index('ix_workflow_runs_organization_id', 'organization_id'),
        Index('ix_workflow_runs_status', 'status'),
        Index('ix_workflow_runs_created_at', 'created_at'),
    )
```

### 1.5 Action and Artifact Models

#### Action Model
```python
class Action(Base):
    __tablename__ = "actions"
    
    # Primary key
    action_id = Column(String, primary_key=True, default=generate_action_id)
    
    # Foreign keys
    step_id = Column(String, ForeignKey("steps.step_id"), nullable=False)
    
    # Action details
    action_type = Column(Enum(ActionType), nullable=False)
    confidence_float = Column(Float, nullable=True)
    
    # Element targeting
    element_id = Column(String, nullable=True)
    text = Column(Text, nullable=True)
    option = Column(JSON, nullable=True)
    file_url = Column(String, nullable=True)
    
    # Coordinates and positioning
    coordinate = Column(JSON, nullable=True)  # [x, y] coordinates
    
    # Reasoning and AI context
    reasoning = Column(Text, nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    # Relationships
    step = relationship("Step", back_populates="actions")
    
    # Indexes
    __table_args__ = (
        Index('ix_actions_step_id', 'step_id'),
        Index('ix_actions_action_type', 'action_type'),
        Index('ix_actions_created_at', 'created_at'),
    )
```

#### Artifact Model
```python
class Artifact(Base):
    __tablename__ = "artifacts"
    
    # Primary key
    artifact_id = Column(String, primary_key=True, default=generate_artifact_id)
    
    # Foreign keys
    task_id = Column(String, ForeignKey("tasks.task_id"), nullable=True)
    step_id = Column(String, ForeignKey("steps.step_id"), nullable=True)
    workflow_run_id = Column(String, ForeignKey("workflow_runs.workflow_run_id"), nullable=True)
    organization_id = Column(String, ForeignKey("organizations.organization_id"), nullable=False)
    
    # Artifact metadata
    artifact_type = Column(Enum(ArtifactType), nullable=False)
    uri = Column(String, nullable=False)
    
    # File information
    signed_url = Column(Text, nullable=True)
    file_size_bytes = Column(BigInteger, nullable=True)
    content_type = Column(String, nullable=True)
    
    # Timestamps
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    # Relationships
    task = relationship("Task", back_populates="artifacts")
    step = relationship("Step", back_populates="artifacts")
    workflow_run = relationship("WorkflowRun", back_populates="artifacts")
    organization = relationship("Organization")
    
    # Indexes
    __table_args__ = (
        Index('ix_artifacts_task_id', 'task_id'),
        Index('ix_artifacts_step_id', 'step_id'),
        Index('ix_artifacts_workflow_run_id', 'workflow_run_id'),
        Index('ix_artifacts_organization_id', 'organization_id'),
        Index('ix_artifacts_artifact_type', 'artifact_type'),
        Index('ix_artifacts_created_at', 'created_at'),
    )
```

---

## 📁 File 2: `skyvern/forge/sdk/db/enums.py` 🔥 **CRITICAL**

### Purpose
Defines all enumeration types used throughout the database schema for consistent status tracking and categorization.

### Core Enumerations

#### Task Status Enumeration
```python
class TaskStatus(str, Enum):
    """Task execution status"""
    created = "created"           # Task created but not started
    queued = "queued"            # Task in execution queue
    running = "running"          # Task currently executing
    completed = "completed"      # Task finished successfully
    failed = "failed"           # Task failed with error
    cancelled = "cancelled"     # Task cancelled by user
    terminated = "terminated"   # Task forcibly terminated
    
    @classmethod
    def terminal_statuses(cls) -> List[str]:
        """Return statuses that indicate task completion"""
        return [cls.completed, cls.failed, cls.cancelled, cls.terminated]
    
    @classmethod
    def active_statuses(cls) -> List[str]:
        """Return statuses that indicate task is active"""
        return [cls.queued, cls.running]
```

#### Step Status Enumeration
```python
class StepStatus(str, Enum):
    """Step execution status"""
    created = "created"          # Step created but not started
    running = "running"          # Step currently executing
    completed = "completed"      # Step finished successfully
    failed = "failed"           # Step failed with error
    retrying = "retrying"       # Step being retried after failure
    skipped = "skipped"         # Step skipped due to conditions
    
    @classmethod
    def can_retry(cls, status: str) -> bool:
        """Check if step status allows retry"""
        return status in [cls.failed, cls.retrying]
```

#### Action Type Enumeration
```python
class ActionType(str, Enum):
    """Browser action types"""
    
    # Navigation actions
    navigate = "navigate"               # Navigate to URL
    go_back = "go_back"                # Browser back button
    go_forward = "go_forward"          # Browser forward button
    refresh = "refresh"                # Refresh page
    
    # Element interaction actions
    click = "click"                    # Click element
    type = "type"                      # Type text into element
    select_option = "select_option"    # Select dropdown option
    check = "check"                    # Check checkbox
    uncheck = "uncheck"               # Uncheck checkbox
    
    # Data actions
    extract = "extract"               # Extract data from page
    upload_file = "upload_file"       # Upload file
    download_file = "download_file"   # Download file
    
    # Waiting actions
    wait = "wait"                     # Wait for condition
    wait_for_element = "wait_for_element"  # Wait for element
    
    # Scroll actions
    scroll = "scroll"                 # Scroll page
    scroll_to_element = "scroll_to_element"  # Scroll to element
    
    # Capture actions
    screenshot = "screenshot"         # Take screenshot
    
    # Completion actions
    complete = "complete"             # Mark task complete
    terminate = "terminate"           # Terminate task
    
    @classmethod
    def interaction_actions(cls) -> List[str]:
        """Return actions that interact with page elements"""
        return [cls.click, cls.type, cls.select_option, cls.check, cls.uncheck]
    
    @classmethod
    def navigation_actions(cls) -> List[str]:
        """Return actions that change page location"""
        return [cls.navigate, cls.go_back, cls.go_forward, cls.refresh]
```

#### Workflow Status Enumeration
```python
class WorkflowRunStatus(str, Enum):
    """Workflow run execution status"""
    created = "created"              # Workflow run created
    queued = "queued"               # Workflow run queued for execution
    running = "running"             # Workflow run executing
    completed = "completed"         # Workflow run completed successfully
    failed = "failed"              # Workflow run failed
    cancelled = "cancelled"        # Workflow run cancelled
    terminated = "terminated"      # Workflow run terminated
    
    @classmethod
    def is_terminal(cls, status: str) -> bool:
        """Check if status is terminal (no further execution)"""
        return status in [cls.completed, cls.failed, cls.cancelled, cls.terminated]
```

#### Artifact Type Enumeration
```python
class ArtifactType(str, Enum):
    """Types of artifacts that can be generated"""
    
    # Screenshots
    screenshot_action = "screenshot_action"      # Action-specific screenshot
    screenshot_llm = "screenshot_llm"           # LLM analysis screenshot
    screenshot_step = "screenshot_step"         # Step completion screenshot
    
    # Recordings
    recording = "recording"                     # Screen recording
    har_file = "har_file"                      # HTTP Archive file
    
    # Data files
    extracted_data = "extracted_data"          # Extracted information
    downloaded_file = "downloaded_file"        # Downloaded file
    
    # Logs
    browser_logs = "browser_logs"              # Browser console logs
    network_logs = "network_logs"              # Network request logs
    
    @classmethod
    def screenshot_types(cls) -> List[str]:
        """Return all screenshot artifact types"""
        return [cls.screenshot_action, cls.screenshot_llm, cls.screenshot_step]
```

---

## 📁 File 3: `skyvern/forge/sdk/db/client.py` 🔥 **CRITICAL**

### Purpose
Database client providing connection management, CRUD operations, and query implementations.

### Database Client Architecture

```python
class DatabaseClient:
    """Main database client for Skyvern operations"""
    
    def __init__(self, database_url: str, **kwargs):
        self.database_url = database_url
        self.engine = self._create_engine(**kwargs)
        self.session_factory = sessionmaker(bind=self.engine)
    
    def _create_engine(self, **kwargs) -> Engine:
        """Create SQLAlchemy engine with optimized settings"""
        
        return create_engine(
            self.database_url,
            # Connection pool settings
            pool_size=20,
            max_overflow=30,
            pool_timeout=30,
            pool_recycle=3600,
            
            # Performance settings
            echo=False,  # Set to True for SQL debugging
            future=True,  # Use SQLAlchemy 2.0 style
            
            # Additional kwargs
            **kwargs
        )
    
    @contextmanager
    def get_session(self) -> Session:
        """Get database session with automatic cleanup"""
        session = self.session_factory()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()
```

### CRUD Operations Matrix

| Entity | Create | Read | Update | Delete | Bulk | Search | Complex Queries |
|--------|--------|------|--------|---------|------|--------|-----------------|
| **Organization** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | Usage analytics |
| **Task** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | Status filtering, pagination |
| **Step** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | Execution history |
| **Workflow** | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | Version management |
| **WorkflowRun** | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | Execution analytics |
| **Action** | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ | Performance analysis |
| **Artifact** | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ | Storage management |

### Core Database Operations

#### Task Operations
```python
class TaskOperations:
    """Task-related database operations"""
    
    def create_task(self, session: Session, task_data: TaskCreate) -> Task:
        """Create a new task"""
        
        task = Task(
            task_id=generate_task_id(),
            organization_id=task_data.organization_id,
            url=task_data.url,
            navigation_goal=task_data.navigation_goal,
            data_extraction_goal=task_data.data_extraction_goal,
            navigation_payload=task_data.navigation_payload,
            status=TaskStatus.created,
            created_by=task_data.created_by
        )
        
        session.add(task)
        session.flush()  # Get the ID without committing
        
        return task
    
    def get_task_by_id(self, session: Session, task_id: str, 
                      include_steps: bool = False,
                      include_artifacts: bool = False) -> Optional[Task]:
        """Get task by ID with optional related data"""
        
        query = session.query(Task).filter(Task.task_id == task_id)
        
        if include_steps:
            query = query.options(joinedload(Task.steps))
        
        if include_artifacts:
            query = query.options(joinedload(Task.artifacts))
        
        return query.first()
    
    def get_tasks_by_organization(self, session: Session, 
                                organization_id: str,
                                status_filter: List[TaskStatus] = None,
                                page: int = 1,
                                page_size: int = 20) -> Tuple[List[Task], int]:
        """Get tasks by organization with filtering and pagination"""
        
        query = session.query(Task).filter(Task.organization_id == organization_id)
        
        if status_filter:
            query = query.filter(Task.status.in_(status_filter))
        
        # Get total count
        total_count = query.count()
        
        # Apply pagination
        offset = (page - 1) * page_size
        tasks = query.order_by(Task.created_at.desc()).offset(offset).limit(page_size).all()
        
        return tasks, total_count
    
    def update_task_status(self, session: Session, task_id: str, 
                          status: TaskStatus, 
                          failure_reason: str = None) -> Task:
        """Update task status"""
        
        task = session.query(Task).filter(Task.task_id == task_id).first()
        if not task:
            raise TaskNotFoundError(f"Task {task_id} not found")
        
        task.status = status
        if failure_reason:
            task.failure_reason = failure_reason
        
        if status in TaskStatus.terminal_statuses():
            task.completed_at = datetime.utcnow()
        
        return task
    
    def get_task_execution_summary(self, session: Session, task_id: str) -> Dict:
        """Get comprehensive task execution summary"""
        
        result = session.execute(
            text("""
            SELECT 
                t.task_id,
                t.status as task_status,
                t.created_at,
                t.completed_at,
                COUNT(s.step_id) as total_steps,
                COUNT(CASE WHEN s.status = 'completed' THEN 1 END) as completed_steps,
                COUNT(CASE WHEN s.status = 'failed' THEN 1 END) as failed_steps,
                COUNT(a.artifact_id) as total_artifacts,
                MAX(s.created_at) as last_step_time
            FROM tasks t
            LEFT JOIN steps s ON t.task_id = s.task_id
            LEFT JOIN artifacts a ON t.task_id = a.task_id
            WHERE t.task_id = :task_id
            GROUP BY t.task_id, t.status, t.created_at, t.completed_at
            """),
            {"task_id": task_id}
        ).fetchone()
        
        if not result:
            raise TaskNotFoundError(f"Task {task_id} not found")
        
        return {
            "task_id": result.task_id,
            "status": result.task_status,
            "created_at": result.created_at,
            "completed_at": result.completed_at,
            "execution_time_seconds": (
                (result.completed_at - result.created_at).total_seconds() 
                if result.completed_at else None
            ),
            "total_steps": result.total_steps or 0,
            "completed_steps": result.completed_steps or 0,
            "failed_steps": result.failed_steps or 0,
            "success_rate": (
                (result.completed_steps / result.total_steps * 100) 
                if result.total_steps > 0 else 0
            ),
            "total_artifacts": result.total_artifacts or 0,
            "last_activity": result.last_step_time
        }
```

#### Workflow Operations
```python
class WorkflowOperations:
    """Workflow-related database operations"""
    
    def create_workflow_run(self, session: Session, 
                          workflow_id: str,
                          organization_id: str,
                          parameters: Dict[str, Any],
                          initiated_by: str = None) -> WorkflowRun:
        """Create a new workflow run"""
        
        # Validate workflow exists
        workflow = session.query(Workflow).filter(
            Workflow.workflow_id == workflow_id,
            Workflow.organization_id == organization_id,
            Workflow.is_active == True
        ).first()
        
        if not workflow:
            raise WorkflowNotFoundError(f"Workflow {workflow_id} not found or inactive")
        
        # Create workflow run
        workflow_run = WorkflowRun(
            workflow_run_id=generate_workflow_run_id(),
            workflow_id=workflow_id,
            organization_id=organization_id,
            initiated_by=initiated_by,
            parameters=parameters,
            status=WorkflowRunStatus.created
        )
        
        session.add(workflow_run)
        session.flush()
        
        return workflow_run
    
    def get_workflow_runs_by_workflow(self, session: Session,
                                    workflow_id: str,
                                    status_filter: List[WorkflowRunStatus] = None,
                                    page: int = 1,
                                    page_size: int = 20) -> Tuple[List[WorkflowRun], int]:
        """Get workflow runs with filtering and pagination"""
        
        query = session.query(WorkflowRun).filter(WorkflowRun.workflow_id == workflow_id)
        
        if status_filter:
            query = query.filter(WorkflowRun.status.in_(status_filter))
        
        total_count = query.count()
        
        offset = (page - 1) * page_size
        runs = query.order_by(WorkflowRun.created_at.desc()).offset(offset).limit(page_size).all()
        
        return runs, total_count
    
    def get_workflow_analytics(self, session: Session, 
                             workflow_id: str,
                             days: int = 30) -> Dict:
        """Get workflow execution analytics"""
        
        cutoff_date = datetime.utcnow() - timedelta(days=days)
        
        result = session.execute(
            text("""
            SELECT 
                COUNT(*) as total_runs,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as successful_runs,
                COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_runs,
                AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_execution_time,
                MIN(created_at) as first_run,
                MAX(created_at) as last_run
            FROM workflow_runs 
            WHERE workflow_id = :workflow_id 
            AND created_at >= :cutoff_date
            """),
            {"workflow_id": workflow_id, "cutoff_date": cutoff_date}
        ).fetchone()
        
        return {
            "workflow_id": workflow_id,
            "period_days": days,
            "total_runs": result.total_runs or 0,
            "successful_runs": result.successful_runs or 0,
            "failed_runs": result.failed_runs or 0,
            "success_rate": (
                (result.successful_runs / result.total_runs * 100) 
                if result.total_runs > 0 else 0
            ),
            "avg_execution_time_seconds": result.avg_execution_time,
            "first_run": result.first_run,
            "last_run": result.last_run
        }

#### Step Operations
```python
class StepOperations:
    """Step-related database operations"""
    
    def create_step(self, session: Session, step_data: StepCreate) -> Step:
        """Create a new step"""
        
        # Get the next order number for this task
        max_order = session.query(func.max(Step.order)).filter(
            Step.task_id == step_data.task_id
        ).scalar() or 0
        
        step = Step(
            step_id=generate_step_id(),
            task_id=step_data.task_id,
            order=max_order + 1,
            step_type=step_data.step_type,
            input_data=step_data.input_data,
            status=StepStatus.created
        )
        
        session.add(step)
        session.flush()
        
        return step
    
    def get_steps_by_task(self, session: Session, 
                         task_id: str,
                         include_actions: bool = False,
                         include_screenshots: bool = False) -> List[Step]:
        """Get all steps for a task ordered by execution order"""
        
        query = session.query(Step).filter(Step.task_id == task_id)
        
        if include_actions:
            query = query.options(joinedload(Step.actions))
        
        if include_screenshots:
            query = query.options(joinedload(Step.screenshots))
        
        return query.order_by(Step.order).all()
    
    def update_step_status(self, session: Session, 
                          step_id: str,
                          status: StepStatus,
                          output_data: Dict = None,
                          failure_reason: str = None) -> Step:
        """Update step status and results"""
        
        step = session.query(Step).filter(Step.step_id == step_id).first()
        if not step:
            raise StepNotFoundError(f"Step {step_id} not found")
        
        step.status = status
        
        if output_data:
            step.output_data = output_data
        
        if failure_reason:
            step.failure_reason = failure_reason
        
        # Update timestamps
        if status == StepStatus.running and not step.started_at:
            step.started_at = datetime.utcnow()
        elif status in [StepStatus.completed, StepStatus.failed]:
            step.completed_at = datetime.utcnow()
        
        return step
    
    def get_step_execution_metrics(self, session: Session, 
                                 organization_id: str,
                                 days: int = 7) -> Dict:
        """Get step execution metrics for organization"""
        
        cutoff_date = datetime.utcnow() - timedelta(days=days)
        
        result = session.execute(
            text("""
            SELECT 
                s.step_type,
                COUNT(*) as total_steps,
                COUNT(CASE WHEN s.status = 'completed' THEN 1 END) as completed_steps,
                COUNT(CASE WHEN s.status = 'failed' THEN 1 END) as failed_steps,
                AVG(EXTRACT(EPOCH FROM (s.completed_at - s.started_at))) as avg_execution_time,
                AVG(s.retry_index) as avg_retries
            FROM steps s
            JOIN tasks t ON s.task_id = t.task_id
            WHERE t.organization_id = :org_id
            AND s.created_at >= :cutoff_date
            GROUP BY s.step_type
            ORDER BY total_steps DESC
            """),
            {"org_id": organization_id, "cutoff_date": cutoff_date}
        ).fetchall()
        
        metrics = {}
        for row in result:
            metrics[row.step_type] = {
                "total_steps": row.total_steps,
                "completed_steps": row.completed_steps,
                "failed_steps": row.failed_steps,
                "success_rate": (row.completed_steps / row.total_steps * 100) if row.total_steps > 0 else 0,
                "avg_execution_time_seconds": row.avg_execution_time,
                "avg_retries": row.avg_retries
            }
        
        return metrics

#### Organization Operations
```python
class OrganizationOperations:
    """Organization-related database operations"""
    
    def create_organization(self, session: Session, org_data: OrganizationCreate) -> Organization:
        """Create a new organization"""
        
        # Check if organization name already exists
        existing = session.query(Organization).filter(
            Organization.organization_name == org_data.organization_name
        ).first()
        
        if existing:
            raise OrganizationExistsError(f"Organization '{org_data.organization_name}' already exists")
        
        organization = Organization(
            organization_id=generate_organization_id(),
            organization_name=org_data.organization_name,
            domain=org_data.domain,
            webhook_callback_url=org_data.webhook_callback_url,
            max_steps_per_run=org_data.max_steps_per_run or 10,
            max_retries_per_step=org_data.max_retries_per_step or 3,
            settings=org_data.settings or {}
        )
        
        session.add(organization)
        session.flush()
        
        return organization
    
    def get_organization_usage_stats(self, session: Session, 
                                   organization_id: str,
                                   start_date: datetime = None,
                                   end_date: datetime = None) -> Dict:
        """Get comprehensive usage statistics for organization"""
        
        if not start_date:
            start_date = datetime.utcnow() - timedelta(days=30)
        if not end_date:
            end_date = datetime.utcnow()
        
        # Task statistics
        task_stats = session.execute(
            text("""
            SELECT 
                COUNT(*) as total_tasks,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed_tasks,
                COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_tasks,
                COUNT(CASE WHEN status IN ('running', 'queued') THEN 1 END) as active_tasks,
                AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_task_duration
            FROM tasks 
            WHERE organization_id = :org_id
            AND created_at BETWEEN :start_date AND :end_date
            """),
            {"org_id": organization_id, "start_date": start_date, "end_date": end_date}
        ).fetchone()
        
        # Workflow statistics  
        workflow_stats = session.execute(
            text("""
            SELECT 
                COUNT(DISTINCT w.workflow_id) as total_workflows,
                COUNT(wr.workflow_run_id) as total_workflow_runs,
                COUNT(CASE WHEN wr.status = 'completed' THEN 1 END) as completed_runs,
                AVG(EXTRACT(EPOCH FROM (wr.completed_at - wr.started_at))) as avg_workflow_duration
            FROM workflows w
            LEFT JOIN workflow_runs wr ON w.workflow_id = wr.workflow_id
            WHERE w.organization_id = :org_id
            AND (wr.created_at IS NULL OR wr.created_at BETWEEN :start_date AND :end_date)
            """),
            {"org_id": organization_id, "start_date": start_date, "end_date": end_date}
        ).fetchone()
        
        # Resource usage
        resource_stats = session.execute(
            text("""
            SELECT 
                COUNT(a.artifact_id) as total_artifacts,
                SUM(a.file_size_bytes) as total_storage_bytes,
                COUNT(CASE WHEN a.artifact_type LIKE 'screenshot%' THEN 1 END) as screenshot_count
            FROM artifacts a
            WHERE a.organization_id = :org_id
            AND a.created_at BETWEEN :start_date AND :end_date
            """),
            {"org_id": organization_id, "start_date": start_date, "end_date": end_date}
        ).fetchone()
        
        return {
            "organization_id": organization_id,
            "period": {
                "start_date": start_date.isoformat(),
                "end_date": end_date.isoformat(),
                "days": (end_date - start_date).days
            },
            "tasks": {
                "total": task_stats.total_tasks or 0,
                "completed": task_stats.completed_tasks or 0,
                "failed": task_stats.failed_tasks or 0,
                "active": task_stats.active_tasks or 0,
                "success_rate": (
                    (task_stats.completed_tasks / task_stats.total_tasks * 100)
                    if task_stats.total_tasks > 0 else 0
                ),
                "avg_duration_seconds": task_stats.avg_task_duration
            },
            "workflows": {
                "total_workflows": workflow_stats.total_workflows or 0,
                "total_runs": workflow_stats.total_workflow_runs or 0,
                "completed_runs": workflow_stats.completed_runs or 0,
                "avg_duration_seconds": workflow_stats.avg_workflow_duration
            },
            "resources": {
                "total_artifacts": resource_stats.total_artifacts or 0,
                "storage_bytes": resource_stats.total_storage_bytes or 0,
                "storage_mb": round((resource_stats.total_storage_bytes or 0) / 1024 / 1024, 2),
                "screenshot_count": resource_stats.screenshot_count or 0
            }
        }
```

---

## 📁 File 4: `skyvern/forge/sdk/schemas/tasks.py` 🔥 **CRITICAL**

### Purpose
Pydantic schemas for task-related data validation, serialization, and API contracts.

### Schema Hierarchy

```mermaid
classDiagram
    class BaseTask {
        +url: str
        +navigation_goal: str
        +data_extraction_goal: Optional[str]
        +navigation_payload: Optional[Dict]
    }
    
    class TaskCreate {
        +organization_id: str
        +created_by: Optional[str]
        +webhook_callback_url: Optional[str]
        +proxy_location: Optional[str]
    }
    
    class TaskUpdate {
        +status: Optional[TaskStatus]
        +extracted_information: Optional[Dict]
        +failure_reason: Optional[str]
    }
    
    class TaskResponse {
        +task_id: str
        +status: TaskStatus
        +created_at: datetime
        +completed_at: Optional[datetime]
        +steps: Optional[List[StepResponse]]
        +artifacts: Optional[List[ArtifactResponse]]
    }
    
    BaseTask <|-- TaskCreate
    BaseTask <|-- TaskUpdate
    BaseTask <|-- TaskResponse
```

### Core Task Schemas

#### Base Task Schema
```python
class BaseTask(BaseModel):
    """Base task schema with common fields"""
    
    url: str = Field(..., description="Target URL for automation")
    navigation_goal: str = Field(..., description="What the automation should accomplish")
    data_extraction_goal: Optional[str] = Field(None, description="Data to extract from the page")
    navigation_payload: Optional[Dict[str, Any]] = Field(None, description="Additional parameters for navigation")
    
    @validator('url')
    def validate_url(cls, v):
        """Validate URL format"""
        if not v.startswith(('http://', 'https://')):
            raise ValueError('URL must start with http:// or https://')
        return v
    
    @validator('navigation_goal')
    def validate_navigation_goal(cls, v):
        """Validate navigation goal is not empty"""
        if not v.strip():
            raise ValueError('Navigation goal cannot be empty')
        return v.strip()
    
    class Config:
        extra = "forbid"  # Prevent additional fields
        str_strip_whitespace = True  # Auto-strip whitespace

class TaskCreate(BaseTask):
    """Schema for creating new tasks"""
    
    organization_id: str = Field(..., description="Organization that owns this task")
    created_by: Optional[str] = Field(None, description="User who created the task")
    webhook_callback_url: Optional[str] = Field(None, description="URL to call when task completes")
    proxy_location: Optional[str] = Field(None, description="Proxy location for browser")
    totp_verification_url: Optional[str] = Field(None, description="URL for TOTP verification")
    totp_identifier: Optional[str] = Field(None, description="Identifier for TOTP verification")
    
    @validator('webhook_callback_url')
    def validate_webhook_url(cls, v):
        """Validate webhook URL if provided"""
        if v and not v.startswith(('http://', 'https://')):
            raise ValueError('Webhook URL must start with http:// or https://')
        return v

class TaskUpdate(BaseModel):
    """Schema for updating existing tasks"""
    
    status: Optional[TaskStatus] = Field(None, description="New task status")
    extracted_information: Optional[Dict[str, Any]] = Field(None, description="Extracted data")
    failure_reason: Optional[str] = Field(None, description="Reason for failure if applicable")
    
    @validator('status')
    def validate_status_transition(cls, v):
        """Validate status transitions are allowed"""
        # Add business logic for valid status transitions
        return v

class TaskResponse(BaseTask):
    """Schema for task API responses"""
    
    task_id: str = Field(..., description="Unique task identifier")
    organization_id: str = Field(..., description="Organization that owns this task")
    status: TaskStatus = Field(..., description="Current task status")
    
    # Timestamps
    created_at: datetime = Field(..., description="When task was created")
    modified_at: Optional[datetime] = Field(None, description="When task was last modified")
    completed_at: Optional[datetime] = Field(None, description="When task completed")
    
    # Results
    extracted_information: Optional[Dict[str, Any]] = Field(None, description="Extracted data")
    failure_reason: Optional[str] = Field(None, description="Failure reason if applicable")
    
    # Related data (optional)
    steps: Optional[List["StepResponse"]] = Field(None, description="Task execution steps")
    artifacts: Optional[List["ArtifactResponse"]] = Field(None, description="Generated artifacts")
    
    class Config:
        from_attributes = True  # Enable ORM mode for SQLAlchemy models

class TaskList(BaseModel):
    """Schema for paginated task lists"""
    
    tasks: List[TaskResponse] = Field(..., description="List of tasks")
    total_count: int = Field(..., description="Total number of tasks")
    page: int = Field(..., description="Current page number")
    page_size: int = Field(..., description="Number of items per page")
    total_pages: int = Field(..., description="Total number of pages")
    
    @validator('total_pages', always=True)
    def calculate_total_pages(cls, v, values):
        """Calculate total pages from total_count and page_size"""
        total_count = values.get('total_count', 0)
        page_size = values.get('page_size', 1)
        return math.ceil(total_count / page_size) if page_size > 0 else 0

class TaskSummary(BaseModel):
    """Schema for task execution summary"""
    
    task_id: str
    status: TaskStatus
    execution_time_seconds: Optional[float]
    total_steps: int
    completed_steps: int
    failed_steps: int
    success_rate: float
    total_artifacts: int
    last_activity: Optional[datetime]
    
    @validator('success_rate')
    def round_success_rate(cls, v):
        """Round success rate to 2 decimal places"""
        return round(v, 2)
```

#### Step Schemas
```python
class StepCreate(BaseModel):
    """Schema for creating new steps"""
    
    task_id: str = Field(..., description="Parent task ID")
    step_type: str = Field(..., description="Type of step (action, validation, extraction)")
    input_data: Optional[Dict[str, Any]] = Field(None, description="Input data for step execution")
    
    @validator('step_type')
    def validate_step_type(cls, v):
        """Validate step type is supported"""
        allowed_types = ["action", "validation", "extraction", "navigation"]
        if v not in allowed_types:
            raise ValueError(f'Step type must be one of: {allowed_types}')
        return v

class StepResponse(BaseModel):
    """Schema for step API responses"""
    
    step_id: str = Field(..., description="Unique step identifier")
    task_id: str = Field(..., description="Parent task ID")
    status: StepStatus = Field(..., description="Current step status")
    order: int = Field(..., description="Execution order within task")
    retry_index: int = Field(..., description="Number of retry attempts")
    
    # Step content
    step_type: str = Field(..., description="Type of step")
    input_data: Optional[Dict[str, Any]] = Field(None, description="Input data")
    output_data: Optional[Dict[str, Any]] = Field(None, description="Output data")
    
    # Results
    is_goal_achieved: Optional[bool] = Field(None, description="Whether step achieved its goal")
    failure_reason: Optional[str] = Field(None, description="Failure reason if applicable")
    
    # Timestamps
    created_at: datetime = Field(..., description="When step was created")
    started_at: Optional[datetime] = Field(None, description="When step execution started")
    completed_at: Optional[datetime] = Field(None, description="When step completed")
    
    # Related data
    actions: Optional[List["ActionResponse"]] = Field(None, description="Actions performed in this step")
    screenshots: Optional[List["ScreenshotResponse"]] = Field(None, description="Screenshots taken")
    
    class Config:
        from_attributes = True
```

#### Action Schemas
```python
class ActionCreate(BaseModel):
    """Schema for creating new actions"""
    
    step_id: str = Field(..., description="Parent step ID")
    action_type: ActionType = Field(..., description="Type of action to perform")
    
    # Element targeting
    element_id: Optional[str] = Field(None, description="Target element ID")
    text: Optional[str] = Field(None, description="Text to input or search for")
    coordinate: Optional[List[float]] = Field(None, description="X,Y coordinates for action")
    
    # Additional parameters
    option: Optional[Dict[str, Any]] = Field(None, description="Additional action parameters")
    file_url: Optional[str] = Field(None, description="File URL for upload actions")
    
    # AI context
    reasoning: Optional[str] = Field(None, description="AI reasoning for this action")
    confidence_float: Optional[float] = Field(None, description="AI confidence score")
    
    @validator('coordinate')
    def validate_coordinate(cls, v):
        """Validate coordinate format"""
        if v is not None:
            if len(v) != 2:
                raise ValueError('Coordinate must be [x, y] format')
            if not all(isinstance(coord, (int, float)) for coord in v):
                raise ValueError('Coordinates must be numeric')
        return v
    
    @validator('confidence_float')
    def validate_confidence(cls, v):
        """Validate confidence score range"""
        if v is not None and not 0.0 <= v <= 1.0:
            raise ValueError('Confidence must be between 0.0 and 1.0')
        return v

class ActionResponse(BaseModel):
    """Schema for action API responses"""
    
    action_id: str = Field(..., description="Unique action identifier")
    step_id: str = Field(..., description="Parent step ID")
    action_type: ActionType = Field(..., description="Type of action")
    
    # Element details
    element_id: Optional[str] = Field(None, description="Target element ID")
    text: Optional[str] = Field(None, description="Text input or target")
    coordinate: Optional[List[float]] = Field(None, description="Action coordinates")
    
    # Results
    option: Optional[Dict[str, Any]] = Field(None, description="Action parameters")
    file_url: Optional[str] = Field(None, description="File URL")
    reasoning: Optional[str] = Field(None, description="AI reasoning")
    confidence_float: Optional[float] = Field(None, description="AI confidence")
    
    # Timestamp
    created_at: datetime = Field(..., description="When action was created")
    
    class Config:
        from_attributes = True
```

---

## 📊 Database Performance Optimization

### Index Strategy

#### Primary Indexes
```sql
-- Task indexes for common queries
CREATE INDEX CONCURRENTLY ix_tasks_organization_status ON tasks(organization_id, status);
CREATE INDEX CONCURRENTLY ix_tasks_created_at_desc ON tasks(created_at DESC);
CREATE INDEX CONCURRENTLY ix_tasks_status_completed_at ON tasks(status, completed_at) WHERE completed_at IS NOT NULL;

-- Step indexes for task relationships
CREATE INDEX CONCURRENTLY ix_steps_task_order ON steps(task_id, "order");
CREATE INDEX CONCURRENTLY ix_steps_status_task ON steps(status, task_id);

-- Workflow indexes for execution tracking
CREATE INDEX CONCURRENTLY ix_workflow_runs_workflow_status ON workflow_runs(workflow_id, status);
CREATE INDEX CONCURRENTLY ix_workflow_runs_org_created ON workflow_runs(organization_id, created_at DESC);

-- Artifact indexes for storage management
CREATE INDEX CONCURRENTLY ix_artifacts_type_created ON artifacts(artifact_type, created_at);
CREATE INDEX CONCURRENTLY ix_artifacts_org_size ON artifacts(organization_id, file_size_bytes);
```

#### Composite Indexes for Complex Queries
```sql
-- Multi-column indexes for common filter combinations
CREATE INDEX CONCURRENTLY ix_tasks_org_status_created ON tasks(organization_id, status, created_at DESC);
CREATE INDEX CONCURRENTLY ix_steps_task_status_order ON steps(task_id, status, "order");
CREATE INDEX CONCURRENTLY ix_workflow_runs_status_started ON workflow_runs(status, started_at) WHERE started_at IS NOT NULL;
```

### Query Optimization Patterns

#### Pagination Best Practices
```python
def get_tasks_paginated_optimized(session: Session,
                                organization_id: str,
                                page: int = 1,
                                page_size: int = 20,
                                status_filter: List[TaskStatus] = None) -> Tuple[List[Task], int]:
    """Optimized pagination using cursor-based approach for large datasets"""
    
    # Base query with proper indexing
    query = session.query(Task).filter(Task.organization_id == organization_id)
    
    if status_filter:
        query = query.filter(Task.status.in_(status_filter))
    
    # Use cursor-based pagination for better performance on large datasets
    if page > 1:
        # Calculate offset
        offset = (page - 1) * page_size
        query = query.offset(offset)
    
    # Optimize with limit and proper ordering
    tasks = query.order_by(Task.created_at.desc()).limit(page_size).all()
    
    # Get count efficiently (only when needed)
    if page == 1:
        # For first page, get exact count
        count_query = session.query(func.count(Task.task_id)).filter(Task.organization_id == organization_id)
        if status_filter:
            count_query = count_query.filter(Task.status.in_(status_filter))
        total_count = count_query.scalar()
    else:
        # For subsequent pages, estimate or use cached count
        total_count = -1  # Indicates estimation needed
    
    return tasks, total_count

#### Bulk Operations
```python
def bulk_update_task_status(session: Session,
                          task_ids: List[str],
                          status: TaskStatus,
                          batch_size: int = 100) -> int:
    """Efficiently update multiple task statuses"""
    
    updated_count = 0
    
    # Process in batches to avoid memory issues
    for i in range(0, len(task_ids), batch_size):
        batch_ids = task_ids[i:i + batch_size]
        
        # Use bulk update for better performance
        result = session.execute(
            update(Task)
            .where(Task.task_id.in_(batch_ids))
            .values(
                status=status,
                modified_at=func.now()
            )
        )
        
        updated_count += result.rowcount
        
        # Commit each batch
        session.commit()
    
    return updated_count

def bulk_create_steps(session: Session,
                     steps_data: List[Dict[str, Any]]) -> List[Step]:
    """Efficiently create multiple steps"""
    
    # Prepare step objects
    steps = []
    for step_data in steps_data:
        step = Step(
            step_id=generate_step_id(),
            **step_data
        )
        steps.append(step)
    
    # Bulk insert
    session.add_all(steps)
    session.flush()  # Get IDs without committing
    
    return steps
```

### Connection Pool Configuration

```python
# Optimized database engine configuration
def create_optimized_engine(database_url: str) -> Engine:
    """Create database engine with production-optimized settings"""
    
    return create_engine(
        database_url,
        
        # Connection pool settings
        pool_size=20,              # Base pool size
        max_overflow=30,           # Additional connections under load
        pool_timeout=30,           # Wait time for connection
        pool_recycle=3600,         # Recycle connections every hour
        pool_pre_ping=True,        # Validate connections before use
        
        # Performance settings
        echo=False,                # Disable SQL logging in production
        future=True,               # Use SQLAlchemy 2.0 features
        connect_args={
            "options": "-c timezone=utc",  # Set timezone
            "application_name": "skyvern",  # For monitoring
            "connect_timeout": 10,          # Connection timeout
        },
        
        # Query optimization
        execution_options={
            "isolation_level": "READ_COMMITTED",
            "autocommit": False
        }
    )
```

---

## 📁 File 5: `skyvern/forge/sdk/db/converters.py` 🔥 **IMPORTANT**

### Purpose
Data transformation utilities for converting between database models and Pydantic schemas.

### Conversion Patterns

```python
class ModelToSchemaConverter:
    """Convert SQLAlchemy models to Pydantic schemas"""
    
    @staticmethod
    def task_to_response(task: Task, 
                        include_steps: bool = False,
                        include_artifacts: bool = False) -> TaskResponse:
        """Convert Task model to TaskResponse schema"""
        
        task_data = {
            "task_id": task.task_id,
            "organization_id": task.organization_id,
            "url": task.url,
            "navigation_goal": task.navigation_goal,
            "data_extraction_goal": task.data_extraction_goal,
            "navigation_payload": task.navigation_payload,
            "status": task.status,
            "created_at": task.created_at,
            "modified_at": task.modified_at,
            "completed_at": task.completed_at,
            "extracted_information": task.extracted_information,
            "failure_reason": task.failure_reason
        }
        
        # Include related data if requested
        if include_steps and task.steps:
            task_data["steps"] = [
                ModelToSchemaConverter.step_to_response(step) 
                for step in task.steps
            ]
        
        if include_artifacts and task.artifacts:
            task_data["artifacts"] = [
                ModelToSchemaConverter.artifact_to_response(artifact) 
                for artifact in task.artifacts
            ]
        
        return TaskResponse(**task_data)
    
    @staticmethod
    def step_to_response(step: Step, 
                        include_actions: bool = False,
                        include_screenshots: bool = False) -> StepResponse:
        """Convert Step model to StepResponse schema"""
        
        step_data = {
            "step_id": step.step_id,
            "task_id": step.task_id,
            "status": step.status,
            "order": step.order,
            "retry_index": step.retry_index,
            "step_type": step.step_type,
            "input_data": step.input_data,
            "output_data": step.output_data,
            "is_goal_achieved": step.is_goal_achieved,
            "failure_reason": step.failure_reason,
            "created_at": step.created_at,
            "started_at": step.started_at,
            "completed_at": step.completed_at
        }
        
        if include_actions and step.actions:
            step_data["actions"] = [
                ModelToSchemaConverter.action_to_response(action) 
                for action in step.actions
            ]
        
        if include_screenshots and step.screenshots:
            step_data["screenshots"] = [
                ModelToSchemaConverter.screenshot_to_response(screenshot) 
                for screenshot in step.screenshots
            ]
        
        return StepResponse(**step_data)
    
    @staticmethod
    def workflow_run_to_response(workflow_run: WorkflowRun,
                               include_blocks: bool = False) -> WorkflowRunResponse:
        """Convert WorkflowRun model to WorkflowRunResponse schema"""
        
        run_data = {
            "workflow_run_id": workflow_run.workflow_run_id,
            "workflow_id": workflow_run.workflow_id,
            "organization_id": workflow_run.organization_id,
            "status": workflow_run.status,
            "current_block_index": workflow_run.current_block_index,
            "parameters": workflow_run.parameters,
            "outputs": workflow_run.outputs,
            "failure_reason": workflow_run.failure_reason,
            "created_at": workflow_run.created_at,
            "started_at": workflow_run.started_at,
            "completed_at": workflow_run.completed_at
        }
        
        if include_blocks and workflow_run.workflow_blocks:
            run_data["blocks"] = [
                ModelToSchemaConverter.workflow_block_to_response(block) 
                for block in workflow_run.workflow_blocks
            ]
        
        return WorkflowRunResponse(**run_data)

class SchemaToModelConverter:
    """Convert Pydantic schemas to SQLAlchemy models"""
    
    @staticmethod
    def task_create_to_model(task_create: TaskCreate) -> Task:
        """Convert TaskCreate schema to Task model"""
        
        return Task(
            task_id=generate_task_id(),
            organization_id=task_create.organization_id,
            url=task_create.url,
            navigation_goal=task_create.navigation_goal,
            data_extraction_goal=task_create.data_extraction_goal,
            navigation_payload=task_create.navigation_payload,
            created_by=task_create.created_by,
            webhook_callback_url=task_create.webhook_callback_url,
            proxy_location=task_create.proxy_location,
            totp_verification_url=task_create.totp_verification_url,
            totp_identifier=task_create.totp_identifier,
            status=TaskStatus.created
        )
    
    @staticmethod
    def step_create_to_model(step_create: StepCreate, order: int) -> Step:
        """Convert StepCreate schema to Step model"""
        
        return Step(
            step_id=generate_step_id(),
            task_id=step_create.task_id,
            order=order,
            step_type=step_create.step_type,
            input_data=step_create.input_data,
            status=StepStatus.created
        )

class DataValidator:
    """Validate data integrity and business rules"""
    
    @staticmethod
    def validate_task_creation(task_data: TaskCreate, session: Session) -> List[str]:
        """Validate task creation data and return any errors"""
        
        errors = []
        
        # Check organization exists
        org = session.query(Organization).filter(
            Organization.organization_id == task_data.organization_id,
            Organization.is_active == True
        ).first()
        
        if not org:
            errors.append(f"Organization {task_data.organization_id} not found or inactive")
        
        # Validate URL accessibility (optional check)
        try:
            import requests
            response = requests.head(task_data.url, timeout=5)
            if response.status_code >= 400:
                errors.append(f"URL {task_data.url} returned status {response.status_code}")
        except Exception as e:
            errors.append(f"URL {task_data.url} is not accessible: {str(e)}")
        
        # Check organization limits
        if org:
            active_tasks = session.query(func.count(Task.task_id)).filter(
                Task.organization_id == task_data.organization_id,
                Task.status.in_(TaskStatus.active_statuses())
            ).scalar()
            
            max_concurrent = org.settings.get("max_concurrent_tasks", 10)
            if active_tasks >= max_concurrent:
                errors.append(f"Organization has reached maximum concurrent tasks ({max_concurrent})")
        
        return errors
    
    @staticmethod
    def validate_workflow_execution(workflow_id: str, 
                                  parameters: Dict[str, Any],
                                  session: Session) -> List[str]:
        """Validate workflow execution parameters"""
        
        errors = []
        
        # Get workflow
        workflow = session.query(Workflow).filter(
            Workflow.workflow_id == workflow_id,
            Workflow.is_active == True
        ).first()
        
        if not workflow:
            errors.append(f"Workflow {workflow_id} not found or inactive")
            return errors
        
        # Validate parameters against schema
        if workflow.parameter_schema:
            try:
                from jsonschema import validate as json_validate
                json_validate(parameters, workflow.parameter_schema)
            except Exception as e:
                errors.append(f"Parameter validation failed: {str(e)}")
        
        return errors
```

---

## 📊 Database Migration Strategy

### Migration Management with Alembic

#### Migration Configuration
```python
# alembic.ini configuration
[alembic]
script_location = migrations
sqlalchemy.url = postgresql+psycopg://user:pass@localhost/skyvern

# Migration environment setup
def run_migrations_online():
    """Run migrations in 'online' mode"""
    
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
            render_as_batch=True  # For SQLite compatibility
        )

        with context.begin_transaction():
            context.run_migrations()
```

#### Schema Evolution Examples

**Migration 001: Initial Schema**
```python
def upgrade():
    """Create initial database schema"""
    
    # Organizations table
    op.create_table(
        'organizations',
        sa.Column('organization_id', sa.String(), nullable=False),
        sa.Column('organization_name', sa.String(), nullable=False),
        sa.Column('domain', sa.String(), nullable=True),
        sa.Column('webhook_callback_url', sa.String(), nullable=True),
        sa.Column('max_steps_per_run', sa.Integer(), nullable=True),
        sa.Column('max_retries_per_step', sa.Integer(), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()')),
        sa.Column('updated_at', sa.DateTime(timezone=True), onupdate=sa.text('now()')),
        sa.Column('is_active', sa.Boolean(), nullable=True),
        sa.Column('settings', sa.JSON(), nullable=True),
        sa.PrimaryKeyConstraint('organization_id'),
        sa.UniqueConstraint('organization_name')
    )
    
    # Tasks table
    op.create_table(
        'tasks',
        sa.Column('task_id', sa.String(), nullable=False),
        sa.Column('organization_id', sa.String(), nullable=False),
        sa.Column('created_by', sa.String(), nullable=True),
        sa.Column('status', sa.Enum(TaskStatus), nullable=True),
        sa.Column('url', sa.Text(), nullable=False),
        sa.Column('navigation_goal', sa.Text(), nullable=False),
        sa.Column('data_extraction_goal', sa.Text(), nullable=True),
        sa.Column('navigation_payload', sa.JSON(), nullable=True),
        sa.Column('extracted_information', sa.JSON(), nullable=True),
        sa.Column('failure_reason', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(timezone=True), server_default=sa.text('now()')),
        sa.Column('modified_at', sa.DateTime(timezone=True), onupdate=sa.text('now()')),
        sa.Column('completed_at', sa.DateTime(timezone=True), nullable=True),
        sa.ForeignKeyConstraint(['organization_id'], ['organizations.organization_id']),
        sa.PrimaryKeyConstraint('task_id')
    )
    
    # Create indexes
    op.create_index('ix_tasks_organization_id', 'tasks', ['organization_id'])
    op.create_index('ix_tasks_status', 'tasks', ['status'])
    op.create_index('ix_tasks_created_at', 'tasks', ['created_at'])
```

**Migration 015: Add TOTP Support**
```python
def upgrade():
    """Add TOTP verification support to tasks"""
    
    # Add TOTP columns to tasks table
    op.add_column('tasks', sa.Column('totp_verification_url', sa.String(), nullable=True))
    op.add_column('tasks', sa.Column('totp_identifier', sa.String(), nullable=True))
    
    # Create index for TOTP lookups
    op.create_index('ix_tasks_totp_identifier', 'tasks', ['totp_identifier'])

def downgrade():
    """Remove TOTP verification support"""
    
    op.drop_index('ix_tasks_totp_identifier', 'tasks')
    op.drop_column('tasks', 'totp_identifier')
    op.drop_column('tasks', 'totp_verification_url')
```

### Data Integrity Constraints

#### Foreign Key Relationships
```sql
-- Ensure referential integrity
ALTER TABLE tasks ADD CONSTRAINT fk_tasks_organization 
    FOREIGN KEY (organization_id) REFERENCES organizations(organization_id) 
    ON DELETE CASCADE;

ALTER TABLE steps ADD CONSTRAINT fk_steps_task 
    FOREIGN KEY (task_id) REFERENCES tasks(task_id) 
    ON DELETE CASCADE;

ALTER TABLE workflow_runs ADD CONSTRAINT fk_workflow_runs_workflow 
    FOREIGN KEY (workflow_id) REFERENCES workflows(workflow_id) 
    ON DELETE CASCADE;
```

#### Check Constraints
```sql
-- Business rule constraints
ALTER TABLE tasks ADD CONSTRAINT check_task_url_format 
    CHECK (url ~* '^https?://.*');

ALTER TABLE organizations ADD CONSTRAINT check_org_limits 
    CHECK (max_steps_per_run > 0 AND max_retries_per_step >= 0);

ALTER TABLE steps ADD CONSTRAINT check_step_order 
    CHECK ("order" > 0);

ALTER TABLE artifacts ADD CONSTRAINT check_file_size 
    CHECK (file_size_bytes >= 0);
```

---

## 🔍 Advanced Query Patterns

### Analytical Queries

#### Organization Performance Analytics
```python
def get_organization_performance_metrics(session: Session,
                                       organization_id: str,
                                       start_date: datetime,
                                       end_date: datetime) -> Dict:
    """Comprehensive organization performance analysis"""
    
    result = session.execute(
        text("""
        WITH task_metrics AS (
            SELECT 
                DATE_TRUNC('day', created_at) as date,
                COUNT(*) as daily_tasks,
                COUNT(CASE WHEN status = 'completed' THEN 1 END) as completed_tasks,
                COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_tasks,
                AVG(EXTRACT(EPOCH FROM (completed_at - created_at))) as avg_duration,
                AVG((
                    SELECT COUNT(*) FROM steps s 
                    WHERE s.task_id = t.task_id AND s.status = 'completed'
                )) as avg_steps_per_task
            FROM tasks t
            WHERE t.organization_id = :org_id
            AND t.created_at BETWEEN :start_date AND :end_date
            GROUP BY DATE_TRUNC('day', created_at)
        ),
        error_analysis AS (
            SELECT 
                failure_reason,
                COUNT(*) as error_count,
                COUNT(*) * 100.0 / (
                    SELECT COUNT(*) FROM tasks 
                    WHERE organization_id = :org_id 
                    AND status = 'failed'
                    AND created_at BETWEEN :start_date AND :end_date
                ) as error_percentage
            FROM tasks
            WHERE organization_id = :org_id
            AND status = 'failed'
            AND created_at BETWEEN :start_date AND :end_date
            AND failure_reason IS NOT NULL
            GROUP BY failure_reason
            ORDER BY error_count DESC
            LIMIT 10
        ),
        resource_usage AS (
            SELECT 
                artifact_type,
                COUNT(*) as artifact_count,
                SUM(file_size_bytes) as total_size_bytes,
                AVG(file_size_bytes) as avg_size_bytes
            FROM artifacts
            WHERE organization_id = :org_id
            AND created_at BETWEEN :start_date AND :end_date
            GROUP BY artifact_type
        )
        SELECT 
            (SELECT json_agg(row_to_json(tm)) FROM task_metrics tm) as daily_metrics,
            (SELECT json_agg(row_to_json(ea)) FROM error_analysis ea) as error_breakdown,
            (SELECT json_agg(row_to_json(ru)) FROM resource_usage ru) as resource_breakdown
        """),
        {
            "org_id": organization_id,
            "start_date": start_date,
            "end_date": end_date
        }
    ).fetchone()
    
    return {
        "organization_id": organization_id,
        "period": {
            "start_date": start_date.isoformat(),
            "end_date": end_date.isoformat()
        },
        "daily_metrics": result.daily_metrics or [],
        "error_breakdown": result.error_breakdown or [],
        "resource_breakdown": result.resource_breakdown or []
    }

#### Workflow Optimization Analysis
```python
def analyze_workflow_bottlenecks(session: Session,
                               workflow_id: str,
                               days: int = 30) -> Dict:
    """Identify bottlenecks in workflow execution"""
    
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    
    result = session.execute(
        text("""
        WITH block_performance AS (
            SELECT 
                wb.block_type,
                wb.block_index,
                COUNT(*) as execution_count,
                AVG(EXTRACT(EPOCH FROM (wb.completed_at - wb.started_at))) as avg_duration,
                PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (wb.completed_at - wb.started_at))) as median_duration,
                PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (wb.completed_at - wb.started_at))) as p95_duration,
                COUNT(CASE WHEN wb.status = 'failed' THEN 1 END) as failure_count,
                COUNT(CASE WHEN wb.status = 'failed' THEN 1 END) * 100.0 / COUNT(*) as failure_rate
            FROM workflow_blocks wb
            JOIN workflow_runs wr ON wb.workflow_run_id = wr.workflow_run_id
            WHERE wr.workflow_id = :workflow_id
            AND wr.created_at >= :cutoff_date
            AND wb.started_at IS NOT NULL
            AND wb.completed_at IS NOT NULL
            GROUP BY wb.block_type, wb.block_index
        ),
        transition_analysis AS (
            SELECT 
                LAG(wb.block_index) OVER (PARTITION BY wb.workflow_run_id ORDER BY wb.block_index) as from_block,
                wb.block_index as to_block,
                COUNT(*) as transition_count,
                AVG(EXTRACT(EPOCH FROM (wb.started_at - LAG(wb.completed_at) OVER (PARTITION BY wb.workflow_run_id ORDER BY wb.block_index)))) as avg_transition_time
            FROM workflow_blocks wb
            JOIN workflow_runs wr ON wb.workflow_run_id = wr.workflow_run_id
            WHERE wr.workflow_id = :workflow_id
            AND wr.created_at >= :cutoff_date
            GROUP BY wb.block_index
            HAVING LAG(wb.block_index) OVER (PARTITION BY wb.workflow_run_id ORDER BY wb.block_index) IS NOT NULL
        )
        SELECT 
            (SELECT json_agg(row_to_json(bp) ORDER BY bp.avg_duration DESC) FROM block_performance bp) as block_performance,
            (SELECT json_agg(row_to_json(ta) ORDER BY ta.avg_transition_time DESC) FROM transition_analysis ta) as transition_analysis
        """),
        {
            "workflow_id": workflow_id,
            "cutoff_date": cutoff_date
        }
    ).fetchone()
    
    return {
        "workflow_id": workflow_id,
        "analysis_period_days": days,
        "block_performance": result.block_performance or [],
        "transition_analysis": result.transition_analysis or [],
        "recommendations": _generate_optimization_recommendations(result)
    }

def _generate_optimization_recommendations(analysis_result) -> List[str]:
    """Generate optimization recommendations based on analysis"""
    
    recommendations = []
    
    if analysis_result.block_performance:
        # Find slowest blocks
        sorted_blocks = sorted(
            analysis_result.block_performance, 
            key=lambda x: x.get('avg_duration', 0), 
            reverse=True
        )
        
        if sorted_blocks and sorted_blocks[0].get('avg_duration', 0) > 30:
            recommendations.append(
                f"Block {sorted_blocks[0]['block_index']} ({sorted_blocks[0]['block_type']}) "
                f"is taking {sorted_blocks[0]['avg_duration']:.1f}s on average. Consider optimization."
            )
        
        # Find high failure rate blocks
        high_failure_blocks = [
            block for block in analysis_result.block_performance 
            if block.get('failure_rate', 0) > 10
        ]
        
        for block in high_failure_blocks:
            recommendations.append(
                f"Block {block['block_index']} has {block['failure_rate']:.1f}% failure rate. "
                f"Review error handling and retry logic."
            )
    
    return recommendations
```

---

## 📊 Database Monitoring and Maintenance

### Performance Monitoring Queries

#### Connection and Query Analysis
```sql
-- Monitor active connections
SELECT 
    datname,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    state_change,
    query
FROM pg_stat_activity 
WHERE datname = 'skyvern'
ORDER BY query_start DESC;

-- Identify slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    stddev_time,
    rows
FROM pg_stat_statements 
WHERE query LIKE '%tasks%' 
ORDER BY total_time DESC 
LIMIT 20;

-- Table size and bloat analysis
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

#### Index Usage Analysis
```sql
-- Index usage statistics
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_tup_read,
    idx_tup_fetch,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes 
ORDER BY idx_scan DESC;

-- Unused indexes (candidates for removal)
SELECT 
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes 
WHERE idx_scan = 0
AND indexname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Maintenance Procedures

#### Automated Cleanup Jobs
```python
class DatabaseMaintenanceJobs:
    """Database maintenance and cleanup operations"""
    
    def __init__(self, session_factory):
        self.session_factory = session_factory
    
    def cleanup_old_artifacts(self, days_old: int = 90) -> int:
        """Remove artifacts older than specified days"""
        
        with self.session_factory() as session:
            cutoff_date = datetime.utcnow() - timedelta(days=days_old)
            
            # Get artifacts to delete
            old_artifacts = session.query(Artifact).filter(
                Artifact.created_at < cutoff_date,
                Artifact.artifact_type.in_([
                    ArtifactType.screenshot_action,
                    ArtifactType.screenshot_step,
                    ArtifactType.browser_logs
                ])
            ).all()
            
            # Delete files from storage
            deleted_count = 0
            for artifact in old_artifacts:
                try:
                    # Delete from file storage
                    self._delete_artifact_file(artifact.uri)
                    
                    # Delete from database
                    session.delete(artifact)
                    deleted_count += 1
                    
                except Exception as e:
                    logger.error(f"Failed to delete artifact {artifact.artifact_id}: {e}")
            
            session.commit()
            return deleted_count
    
    def cleanup_completed_tasks(self, days_old: int = 180) -> int:
        """Archive old completed tasks"""
        
        with self.session_factory() as session:
            cutoff_date = datetime.utcnow() - timedelta(days=days_old)
            
            # Move to archive table (if implemented)
            result = session.execute(
                text("""
                WITH archived_tasks AS (
                    INSERT INTO tasks_archive 
                    SELECT * FROM tasks 
                    WHERE status IN ('completed', 'failed', 'cancelled')
                    AND completed_at < :cutoff_date
                    RETURNING task_id
                )
                DELETE FROM tasks 
                WHERE task_id IN (SELECT task_id FROM archived_tasks)
                """),
                {"cutoff_date": cutoff_date}
            )
            
            session.commit()
            return result.rowcount
    
    def optimize_database(self) -> Dict[str, Any]:
        """Run database optimization procedures"""
        
        with self.session_factory() as session:
            results = {}
            
            # Update table statistics
            session.execute(text("ANALYZE;"))
            results["analyze_completed"] = True
            
            # Vacuum specific tables
            large_tables = ["tasks", "steps", "actions", "artifacts", "workflow_runs"]
            for table in large_tables:
                session.execute(text(f"VACUUM {table};"))
            
            results["vacuum_completed"] = large_tables
            
            # Reindex if needed
            session.execute(text("REINDEX DATABASE skyvern;"))
            results["reindex_completed"] = True
            
            session.commit()
            return results
```

---

## ✅ Phase 2 Completion Checklist

### Understanding Verification

- [ ] **Entity Relationships**: Can draw and explain the complete ER diagram
- [ ] **Database Models**: Understand all SQLAlchemy model definitions and relationships
- [ ] **Status Lifecycles**: Know the status transitions for tasks, steps, and workflows
- [ ] **Index Strategy**: Understand how indexes are designed for query optimization
- [ ] **Schema Validation**: Know how Pydantic schemas validate and transform data
- [ ] **Query Patterns**: Can write efficient queries for common operations

### Practical Skills

- [ ] **Database Operations**: Can perform CRUD operations using the database client
- [ ] **Performance Optimization**: Can identify and resolve performance bottlenecks
- [ ] **Data Migration**: Can create and manage database migrations
- [ ] **Monitoring**: Can monitor database performance and identify issues
- [ ] **Maintenance**: Can perform routine database maintenance tasks

### Key Questions to Answer

1. **How are task executions tracked through the database?**
   - Tasks are broken down into Steps, which contain Actions
   - Each step tracks execution status, timing, and results
   - Artifacts store generated files and screenshots
   - All entities maintain audit trails with timestamps

2. **What are the critical indexes for performance?**
   - Composite indexes on (organization_id, status, created_at)
   - Foreign key indexes for all relationships
   - Status-based indexes for filtering operations
   - Time-based indexes for analytics queries

3. **How does the schema handle data validation?**
   - Pydantic schemas validate API inputs and outputs
   - Database constraints enforce data integrity
   - Custom validators check business rules
   - Converters transform between models and schemas

4. **What are the scaling considerations?**
   - Connection pooling for concurrent access
   - Partitioning strategies for large tables
   - Archive procedures for old data
   - Index optimization for query performance

---

## 🚀 Ready for Phase 3?

You now have comprehensive knowledge of Skyvern's database layer, including:

- **Complete entity model** with all relationships and constraints
- **Query patterns** for efficient data access and analytics
- **Performance optimization** strategies and monitoring
- **Schema management** with migrations and validation
- **Maintenance procedures** for production environments

**Next Phase Preview**: Phase 3 will explore the **API & Web Framework**, covering:
- FastAPI application architecture
- REST endpoint design and implementation
- Authentication and security mechanisms
- Request/response handling patterns
- API documentation and testing

The database knowledge you've gained is essential for understanding how the API layer interacts with persistent data.

Type "ready for phase 3" when you want to continue with the API & Web Framework deep dive!