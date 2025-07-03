# Phase 3: API & Web Framework
## Skyvern Repository Deep Dive

---

## 📋 Phase Overview

**Phase 3 Focus**: FastAPI-based REST API providing endpoints for task management, workflow execution, and system administration.

**Timeline**: Week 2 of the 12-phase learning plan

**Prerequisites**: 
- Phase 1 (Foundation & Configuration) ✅
- Phase 2 (Database Layer) ✅

**Importance**: The API layer serves as the primary interface between external clients and Skyvern's automation capabilities.

---

## 🎯 Learning Goals for Phase 3

- ✅ **API Architecture**: Understand FastAPI application structure and design patterns
- ✅ **Endpoint Design**: Know all REST endpoints and their functionality
- ✅ **Authentication**: Grasp API key authentication and security mechanisms
- ✅ **Request Processing**: Understand request validation and response formatting
- ✅ **Error Handling**: Know comprehensive error handling and status code mapping
- ✅ **Performance**: Understand caching, rate limiting, and optimization strategies

---

## 🗂️ Critical Files to Study (Priority Order)

### 🔥 **CRITICAL** Files (Must Master)
1. **`skyvern/forge/app.py`** - FastAPI application initialization
2. **`skyvern/forge/api/endpoints/tasks.py`** - Task management endpoints
3. **`skyvern/forge/api/endpoints/workflows.py`** - Workflow management APIs
4. **`skyvern/forge/sdk/services/org_auth_service.py`** - Authentication service

### 🔶 **IMPORTANT** Files (Should Understand)
5. **`skyvern/forge/api/endpoints/browser_sessions.py`** - Browser session management
6. **`skyvern/forge/api/endpoints/artifacts.py`** - File and artifact management
7. **`skyvern/forge/api/models.py`** - API-specific data models
8. **`skyvern/forge/sdk/core/security.py`** - Security utilities

---

## 🏗️ API Architecture Overview

### FastAPI Application Stack

```mermaid
graph TD
    A[External Clients] --> B[Load Balancer/Nginx]
    B --> C[FastAPI Application]
    C --> D[Middleware Stack]
    D --> E[Route Handlers]
    E --> F[Service Layer]
    F --> G[Database Layer]
    F --> H[Automation Engine]
    
    D --> I[CORS Middleware]
    D --> J[Authentication Middleware]
    D --> K[Rate Limiting Middleware]
    D --> L[Logging Middleware]
    D --> M[Error Handling Middleware]
    
    E --> N[Task Endpoints]
    E --> O[Workflow Endpoints]
    E --> P[Organization Endpoints]
    E --> Q[Browser Session Endpoints]
    E --> R[Artifact Endpoints]
```

### Technology Stack Components

| Layer | Technology | Version | Purpose | Configuration |
|-------|------------|---------|---------|---------------|
| **Web Framework** | FastAPI | 0.104+ | Async REST API framework | High-performance, auto-documentation |
| **ASGI Server** | Uvicorn | 0.24+ | ASGI application server | Production-grade async server |
| **Validation** | Pydantic | 2.0+ | Data validation and serialization | Type-safe request/response models |
| **Authentication** | Custom JWT/API Key | - | Organization-based auth | API key authentication |
| **Documentation** | OpenAPI/Swagger | Auto-generated | Interactive API docs | Built-in FastAPI feature |
| **Monitoring** | Prometheus/Custom | - | Metrics collection | Request/response metrics |

---

## 📁 File 1: `skyvern/forge/app.py` 🔥 **CRITICAL**

### Purpose
Central FastAPI application initialization, middleware configuration, and dependency injection setup.

### Application Architecture

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.responses import JSONResponse
import time
import uuid
from typing import Dict, Any

class SkyvernAPI:
    """Main Skyvern FastAPI application class"""
    
    def __init__(self, settings: Settings):
        self.settings = settings
        self.app = self._create_app()
        self._setup_middleware()
        self._setup_exception_handlers()
        self._setup_routes()
        self._setup_lifecycle_events()
    
    def _create_app(self) -> FastAPI:
        """Create FastAPI application with metadata"""
        
        return FastAPI(
            title="Skyvern API",
            description="AI-powered web automation platform",
            version="2.0.0",
            docs_url="/docs" if self.settings.ENV != "production" else None,
            redoc_url="/redoc" if self.settings.ENV != "production" else None,
            openapi_url="/openapi.json" if self.settings.ENV != "production" else None,
            
            # OpenAPI metadata
            contact={
                "name": "Skyvern Team",
                "url": "https://skyvern.com",
                "email": "support@skyvern.com"
            },
            license_info={
                "name": "MIT License",
                "url": "https://opensource.org/licenses/MIT"
            },
            
            # Performance settings
            generate_unique_id_function=self._generate_unique_operation_id
        )
    
    def _generate_unique_operation_id(self, route) -> str:
        """Generate unique operation IDs for OpenAPI"""
        return f"{route.tags[0]}_{route.name}" if route.tags else route.name
```

### Middleware Stack Configuration

```python
def _setup_middleware(self):
    """Configure middleware stack in correct order"""
    
    # 1. CORS - Must be first for browser requests
    self.app.add_middleware(
        CORSMiddleware,
        allow_origins=self.settings.ALLOWED_ORIGINS,
        allow_credentials=True,
        allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
        allow_headers=["*"],
        expose_headers=["X-Request-ID", "X-Rate-Limit-Remaining"]
    )
    
    # 2. Compression - Before other processing
    self.app.add_middleware(
        GZipMiddleware,
        minimum_size=1000,
        compresslevel=6
    )
    
    # 3. Request ID - For tracing and debugging
    @self.app.middleware("http")
    async def add_request_id(request: Request, call_next):
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        
        response = await call_next(request)
        response.headers["X-Request-ID"] = request_id
        return response
    
    # 4. Timing - Performance monitoring
    @self.app.middleware("http")
    async def add_process_time(request: Request, call_next):
        start_time = time.time()
        response = await call_next(request)
        process_time = time.time() - start_time
        response.headers["X-Process-Time"] = str(process_time)
        return response
    
    # 5. Rate Limiting - Protect against abuse
    @self.app.middleware("http")
    async def rate_limiting(request: Request, call_next):
        # Skip rate limiting for health checks
        if request.url.path in ["/health", "/metrics"]:
            return await call_next(request)
        
        # Extract API key for rate limiting
        api_key = request.headers.get("x-api-key")
        if api_key:
            rate_limit_result = await self._check_rate_limit(api_key, request.url.path)
            if not rate_limit_result.allowed:
                return JSONResponse(
                    status_code=429,
                    content={"error": "Rate limit exceeded", "retry_after": rate_limit_result.retry_after},
                    headers={"Retry-After": str(rate_limit_result.retry_after)}
                )
        
        response = await call_next(request)
        return response
    
    # 6. Logging - Request/response logging
    @self.app.middleware("http")
    async def request_logging(request: Request, call_next):
        # Log request
        logger.info(
            "Request started",
            extra={
                "request_id": getattr(request.state, "request_id", None),
                "method": request.method,
                "url": str(request.url),
                "user_agent": request.headers.get("user-agent"),
                "api_key_hash": self._hash_api_key(request.headers.get("x-api-key"))
            }
        )
        
        response = await call_next(request)
        
        # Log response
        logger.info(
            "Request completed",
            extra={
                "request_id": getattr(request.state, "request_id", None),
                "status_code": response.status_code,
                "response_time": response.headers.get("X-Process-Time")
            }
        )
        
        return response
```

### Middleware Processing Flow

```mermaid
sequenceDiagram
    participant Client
    participant CORS
    participant Compression
    participant RequestID
    participant Timing
    participant RateLimit
    participant Logging
    participant Auth
    participant Handler
    participant Response
    
    Client->>CORS: HTTP Request
    CORS->>Compression: Add CORS headers
    Compression->>RequestID: Setup compression
    RequestID->>Timing: Generate request ID
    Timing->>RateLimit: Start timing
    RateLimit->>Logging: Check rate limits
    Logging->>Auth: Log request
    Auth->>Handler: Authenticate
    Handler->>Response: Process request
    Response->>Logging: Generate response
    Logging->>RateLimit: Log response
    RateLimit->>Timing: Add rate limit headers
    Timing->>RequestID: Add timing header
    RequestID->>Compression: Add request ID
    Compression->>CORS: Compress response
    CORS->>Client: Final response
```

### Exception Handling

```python
def _setup_exception_handlers(self):
    """Setup global exception handlers"""
    
    @self.app.exception_handler(SkyvernException)
    async def skyvern_exception_handler(request: Request, exc: SkyvernException):
        """Handle Skyvern-specific exceptions"""
        
        return JSONResponse(
            status_code=exc.http_status_code,
            content={
                "success": False,
                "error": {
                    "code": exc.error_code,
                    "message": exc.message,
                    "details": exc.details
                },
                "metadata": {
                    "request_id": getattr(request.state, "request_id", None),
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
    
    @self.app.exception_handler(ValidationError)
    async def validation_exception_handler(request: Request, exc: ValidationError):
        """Handle Pydantic validation errors"""
        
        return JSONResponse(
            status_code=422,
            content={
                "success": False,
                "error": {
                    "code": "VALIDATION_ERROR",
                    "message": "Request validation failed",
                    "details": exc.errors()
                },
                "metadata": {
                    "request_id": getattr(request.state, "request_id", None),
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
    
    @self.app.exception_handler(HTTPException)
    async def http_exception_handler(request: Request, exc: HTTPException):
        """Handle FastAPI HTTP exceptions"""
        
        return JSONResponse(
            status_code=exc.status_code,
            content={
                "success": False,
                "error": {
                    "code": f"HTTP_{exc.status_code}",
                    "message": exc.detail,
                    "details": {}
                },
                "metadata": {
                    "request_id": getattr(request.state, "request_id", None),
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
    
    @self.app.exception_handler(Exception)
    async def general_exception_handler(request: Request, exc: Exception):
        """Handle unexpected exceptions"""
        
        # Log the full exception for debugging
        logger.exception(
            "Unhandled exception occurred",
            extra={
                "request_id": getattr(request.state, "request_id", None),
                "exception_type": type(exc).__name__,
                "exception_message": str(exc)
            }
        )
        
        return JSONResponse(
            status_code=500,
            content={
                "success": False,
                "error": {
                    "code": "INTERNAL_SERVER_ERROR",
                    "message": "An unexpected error occurred",
                    "details": {} if self.settings.ENV == "production" else {"exception": str(exc)}
                },
                "metadata": {
                    "request_id": getattr(request.state, "request_id", None),
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
```

### Dependency Injection System

```python
class Dependencies:
    """Dependency injection container"""
    
    @staticmethod
    async def get_database():
        """Get database session dependency"""
        async with AsyncSession(engine) as session:
            try:
                yield session
                await session.commit()
            except Exception:
                await session.rollback()
                raise
            finally:
                await session.close()
    
    @staticmethod
    async def get_current_organization(
        request: Request,
        db: AsyncSession = Depends(get_database)
    ) -> Organization:
        """Get current organization from API key"""
        
        api_key = request.headers.get("x-api-key")
        if not api_key:
            raise HTTPException(
                status_code=401,
                detail="API key required"
            )
        
        # Validate API key and get organization
        auth_service = OrganizationAuthService(db)
        organization = await auth_service.authenticate_api_key(api_key)
        
        if not organization:
            raise HTTPException(
                status_code=401,
                detail="Invalid API key"
            )
        
        return organization
    
    @staticmethod
    async def get_request_context(
        request: Request,
        organization: Organization = Depends(get_current_organization)
    ) -> RequestContext:
        """Get complete request context"""
        
        return RequestContext(
            request_id=getattr(request.state, "request_id", None),
            organization_id=organization.organization_id,
            user_agent=request.headers.get("user-agent"),
            client_ip=request.client.host,
            request_time=datetime.utcnow()
        )

# Apply dependencies globally
app.dependency_overrides = {
    "database": Dependencies.get_database,
    "organization": Dependencies.get_current_organization,
    "context": Dependencies.get_request_context
}
```

### Application Lifecycle Events

```python
def _setup_lifecycle_events(self):
    """Setup application startup and shutdown events"""
    
    @self.app.on_event("startup")
    async def startup_event():
        """Initialize application components on startup"""
        
        logger.info("Starting Skyvern API application")
        
        # Initialize database connections
        await self._initialize_database()
        
        # Initialize browser manager
        await self._initialize_browser_manager()
        
        # Initialize AI/LLM clients
        await self._initialize_ai_clients()
        
        # Initialize background tasks
        await self._start_background_tasks()
        
        # Health check endpoint
        await self._setup_health_checks()
        
        logger.info("Skyvern API application started successfully")
    
    @self.app.on_event("shutdown")
    async def shutdown_event():
        """Cleanup application components on shutdown"""
        
        logger.info("Shutting down Skyvern API application")
        
        # Stop background tasks
        await self._stop_background_tasks()
        
        # Close browser sessions
        await self._cleanup_browser_sessions()
        
        # Close database connections
        await self._cleanup_database()
        
        logger.info("Skyvern API application shutdown complete")
    
    async def _initialize_database(self):
        """Initialize database connections and verify schema"""
        
        try:
            # Test database connectivity
            async with AsyncSession(engine) as session:
                await session.execute(text("SELECT 1"))
            
            # Run pending migrations if in development
            if self.settings.ENV == "development":
                await self._run_migrations()
            
            logger.info("Database initialized successfully")
            
        except Exception as e:
            logger.error(f"Database initialization failed: {e}")
            raise
    
    async def _initialize_browser_manager(self):
        """Initialize browser management system"""
        
        try:
            from skyvern.webeye.browser_manager import BrowserManager
            
            self.browser_manager = BrowserManager(self.settings)
            await self.browser_manager.initialize()
            
            logger.info("Browser manager initialized successfully")
            
        except Exception as e:
            logger.error(f"Browser manager initialization failed: {e}")
            raise
```

---

## 📁 File 2: `skyvern/forge/api/endpoints/tasks.py` 🔥 **CRITICAL**

### Purpose
Task management endpoints for creating, monitoring, and controlling automation tasks.

### Task API Endpoint Architecture

```mermaid
graph TD
    A[Task API Endpoints] --> B[Create Task]
    A --> C[Get Task]
    A --> D[List Tasks]
    A --> E[Update Task]
    A --> F[Cancel Task]
    A --> G[Get Task Steps]
    A --> H[Get Task Artifacts]
    A --> I[Get Task Logs]
    
    B --> J[Validation]
    B --> K[Database Creation]
    B --> L[Queue for Execution]
    
    C --> M[Authentication]
    C --> N[Database Lookup]
    C --> O[Response Formatting]
    
    D --> P[Filtering]
    D --> Q[Pagination]
    D --> R[Sorting]
```

### Core Task Endpoints

#### 1. Create Task Endpoint

```python
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from fastapi.responses import JSONResponse
from typing import Optional, List
import asyncio

router = APIRouter(prefix="/api/v1/tasks", tags=["tasks"])

@router.post("/", response_model=TaskResponse, status_code=201)
async def create_task(
    task_data: TaskCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> TaskResponse:
    """
    Create a new automation task
    
    Creates a new task and queues it for execution by the automation engine.
    The task will be processed asynchronously.
    
    **Parameters:**
    - **url**: Target website URL
    - **navigation_goal**: What the automation should accomplish
    - **data_extraction_goal**: (Optional) Data to extract from the page
    - **navigation_payload**: (Optional) Additional parameters
    
    **Returns:**
    - Task object with ID and status
    """
    
    # Validate task creation request
    validation_errors = await TaskValidator.validate_create_request(
        task_data, organization, db
    )
    if validation_errors:
        raise HTTPException(
            status_code=400,
            detail={
                "code": "VALIDATION_FAILED",
                "message": "Task validation failed",
                "errors": validation_errors
            }
        )
    
    try:
        # Create task in database
        task_service = TaskService(db)
        task = await task_service.create_task(
            task_data=task_data,
            organization_id=organization.organization_id,
            created_by=context.user_id,
            request_id=context.request_id
        )
        
        # Queue task for execution
        background_tasks.add_task(
            queue_task_for_execution,
            task.task_id,
            organization.organization_id
        )
        
        # Send webhook notification if configured
        if task_data.webhook_callback_url:
            background_tasks.add_task(
                send_webhook_notification,
                task_data.webhook_callback_url,
                "task.created",
                {"task_id": task.task_id}
            )
        
        # Convert to response model
        response = ModelToSchemaConverter.task_to_response(task)
        
        # Log task creation
        logger.info(
            "Task created successfully",
            extra={
                "task_id": task.task_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "url": task_data.url
            }
        )
        
        return JSONResponse(
            status_code=201,
            content={
                "success": True,
                "data": response.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except OrganizationLimitExceededError as e:
        raise HTTPException(
            status_code=429,
            detail={
                "code": "LIMIT_EXCEEDED",
                "message": str(e),
                "retry_after": 3600
            }
        )
    
    except Exception as e:
        logger.error(
            "Task creation failed",
            extra={
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "TASK_CREATION_FAILED",
                "message": "Failed to create task"
            }
        )

class TaskValidator:
    """Task validation logic"""
    
    @staticmethod
    async def validate_create_request(
        task_data: TaskCreate,
        organization: Organization,
        db: AsyncSession
    ) -> List[str]:
        """Validate task creation request"""
        
        errors = []
        
        # URL validation
        if not task_data.url.startswith(('http://', 'https://')):
            errors.append("URL must start with http:// or https://")
        
        # Check URL accessibility
        try:
            async with aiohttp.ClientSession() as session:
                async with session.head(task_data.url, timeout=5) as response:
                    if response.status >= 500:
                        errors.append(f"Target URL returned server error: {response.status}")
        except asyncio.TimeoutError:
            errors.append("Target URL is not accessible (timeout)")
        except Exception as e:
            errors.append(f"Target URL validation failed: {str(e)}")
        
        # Organization limits
        active_tasks = await TaskService.get_active_task_count(
            organization.organization_id, db
        )
        
        max_concurrent = organization.settings.get("max_concurrent_tasks", 10)
        if active_tasks >= max_concurrent:
            errors.append(f"Maximum concurrent tasks limit reached ({max_concurrent})")
        
        # Goal validation
        if not task_data.navigation_goal.strip():
            errors.append("Navigation goal cannot be empty")
        
        if len(task_data.navigation_goal) > 2000:
            errors.append("Navigation goal too long (max 2000 characters)")
        
        return errors
```

#### 2. Get Task Endpoint

```python
@router.get("/{task_id}", response_model=TaskResponse)
async def get_task(
    task_id: str,
    include_steps: bool = False,
    include_artifacts: bool = False,
    include_logs: bool = False,
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> TaskResponse:
    """
    Get task details by ID
    
    Retrieves detailed information about a specific task, including
    optional related data like steps, artifacts, and logs.
    
    **Parameters:**
    - **task_id**: Unique task identifier
    - **include_steps**: Include execution steps in response
    - **include_artifacts**: Include generated artifacts
    - **include_logs**: Include execution logs
    
    **Returns:**
    - Complete task information with optional related data
    """
    
    try:
        task_service = TaskService(db)
        task = await task_service.get_task_by_id(
            task_id=task_id,
            organization_id=organization.organization_id,
            include_steps=include_steps,
            include_artifacts=include_artifacts,
            include_logs=include_logs
        )
        
        if not task:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "TASK_NOT_FOUND",
                    "message": f"Task {task_id} not found"
                }
            )
        
        # Convert to response model
        response = ModelToSchemaConverter.task_to_response(
            task,
            include_steps=include_steps,
            include_artifacts=include_artifacts,
            include_logs=include_logs
        )
        
        return JSONResponse(
            content={
                "success": True,
                "data": response.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except TaskAccessDeniedError:
        raise HTTPException(
            status_code=403,
            detail={
                "code": "ACCESS_DENIED",
                "message": "Task access denied for this organization"
            }
        )
    
    except Exception as e:
        logger.error(
            "Get task failed",
            extra={
                "task_id": task_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "GET_TASK_FAILED",
                "message": "Failed to retrieve task"
            }
        )
```

#### 3. List Tasks Endpoint

```python
@router.get("/", response_model=TaskListResponse)
async def list_tasks(
    page: int = Query(1, ge=1, description="Page number (1-based)"),
    page_size: int = Query(20, ge=1, le=100, description="Items per page"),
    status: Optional[List[TaskStatus]] = Query(None, description="Filter by status"),
    created_after: Optional[datetime] = Query(None, description="Filter by creation date"),
    created_before: Optional[datetime] = Query(None, description="Filter by creation date"),
    search: Optional[str] = Query(None, description="Search in navigation goal"),
    sort_by: TaskSortField = Query(TaskSortField.created_at, description="Sort field"),
    sort_order: SortOrder = Query(SortOrder.desc, description="Sort order"),
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> TaskListResponse:
    """
    List tasks with filtering and pagination
    
    Retrieves a paginated list of tasks for the organization with
    comprehensive filtering and sorting options.
    
    **Filtering Options:**
    - **status**: Filter by task status (can specify multiple)
    - **created_after/before**: Date range filtering
    - **search**: Text search in navigation goals
    
    **Sorting Options:**
    - **sort_by**: Field to sort by (created_at, status, url)
    - **sort_order**: Ascending or descending order
    
    **Returns:**
    - Paginated list of tasks with metadata
    """
    
    try:
        task_service = TaskService(db)
        
        # Build filter criteria
        filter_criteria = TaskFilterCriteria(
            organization_id=organization.organization_id,
            status_filter=status,
            created_after=created_after,
            created_before=created_before,
            search_query=search,
            page=page,
            page_size=page_size,
            sort_by=sort_by,
            sort_order=sort_order
        )
        
        # Get tasks and total count
        tasks, total_count = await task_service.list_tasks(filter_criteria)
        
        # Convert to response models
        task_responses = [
            ModelToSchemaConverter.task_to_response(task) 
            for task in tasks
        ]
        
        # Calculate pagination metadata
        total_pages = math.ceil(total_count / page_size)
        has_next = page < total_pages
        has_prev = page > 1
        
        return JSONResponse(
            content={
                "success": True,
                "data": {
                    "tasks": [task.dict() for task in task_responses],
                    "pagination": {
                        "page": page,
                        "page_size": page_size,
                        "total_count": total_count,
                        "total_pages": total_pages,
                        "has_next": has_next,
                        "has_prev": has_prev
                    },
                    "filters": {
                        "status": status,
                        "created_after": created_after.isoformat() if created_after else None,
                        "created_before": created_before.isoformat() if created_before else None,
                        "search": search,
                        "sort_by": sort_by,
                        "sort_order": sort_order
                    }
                },
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "List tasks failed",
            extra={
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "LIST_TASKS_FAILED",
                "message": "Failed to retrieve tasks"
            }
        )

class TaskFilterCriteria(BaseModel):
    """Task filtering and pagination criteria"""
    
    organization_id: str
    status_filter: Optional[List[TaskStatus]] = None
    created_after: Optional[datetime] = None
    created_before: Optional[datetime] = None
    search_query: Optional[str] = None
    page: int = 1
    page_size: int = 20
    sort_by: TaskSortField = TaskSortField.created_at
    sort_order: SortOrder = SortOrder.desc
    
    @validator('page_size')
    def validate_page_size(cls, v):
        if v > 100:
            raise ValueError('Page size cannot exceed 100')
        return v

class TaskSortField(str, Enum):
    """Available fields for task sorting"""
    created_at = "created_at"
    modified_at = "modified_at"
    completed_at = "completed_at"
    status = "status"
    url = "url"
    navigation_goal = "navigation_goal"

class SortOrder(str, Enum):
    """Sort order options"""
    asc = "asc"
    desc = "desc"
```

#### 4. Cancel Task Endpoint

```python
@router.post("/{task_id}/cancel", response_model=TaskResponse)
async def cancel_task(
    task_id: str,
    reason: Optional[str] = Body(None, description="Reason for cancellation"),
    force: bool = Body(False, description="Force cancellation even if task is running"),
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> TaskResponse:
    """
    Cancel a running or queued task
    
    Attempts to gracefully cancel a task. If the task is currently executing,
    it will be marked for cancellation and stopped at the next safe point.
    
    **Parameters:**
    - **task_id**: Task to cancel
    - **reason**: Optional reason for cancellation
    - **force**: Force immediate cancellation (may leave resources in inconsistent state)
    
    **Returns:**
    - Updated task with cancelled status
    """
    
    try:
        task_service = TaskService(db)
        task = await task_service.get_task_by_id(
            task_id=task_id,
            organization_id=organization.organization_id
        )
        
        if not task:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "TASK_NOT_FOUND",
                    "message": f"Task {task_id} not found"
                }
            )
        
        # Check if task can be cancelled
        if task.status in [TaskStatus.completed, TaskStatus.failed, TaskStatus.cancelled]:
            raise HTTPException(
                status_code=409,
                detail={
                    "code": "TASK_ALREADY_FINISHED",
                    "message": f"Task is already {task.status}"
                }
            )
        
        # Cancel the task
        cancellation_result = await task_service.cancel_task(
            task_id=task_id,
            reason=reason,
            force=force,
            cancelled_by=context.user_id
        )
        
        if not cancellation_result.success:
            raise HTTPException(
                status_code=500,
                detail={
                    "code": "CANCELLATION_FAILED",
                    "message": cancellation_result.error_message
                }
            )
        
        # Get updated task
        updated_task = await task_service.get_task_by_id(
            task_id=task_id,
            organization_id=organization.organization_id
        )
        
        response = ModelToSchemaConverter.task_to_response(updated_task)
        
        logger.info(
            "Task cancelled successfully",
            extra={
                "task_id": task_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "reason": reason,
                "force": force
            }
        )
        
        return JSONResponse(
            content={
                "success": True,
                "data": response.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat(),
                    "cancellation_time_ms": cancellation_result.cancellation_time_ms
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "Task cancellation failed",
            extra={
                "task_id": task_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "CANCEL_TASK_FAILED",
                "message": "Failed to cancel task"
            }
        )

#### 5. Get Task Steps Endpoint

```python
@router.get("/{task_id}/steps", response_model=List[StepResponse])
async def get_task_steps(
    task_id: str,
    include_actions: bool = Query(False, description="Include actions in steps"),
    include_screenshots: bool = Query(False, description="Include screenshots"),
    status_filter: Optional[List[StepStatus]] = Query(None, description="Filter by step status"),
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> List[StepResponse]:
    """
    Get execution steps for a task
    
    Retrieves all execution steps for a task in chronological order,
    with optional filtering and related data inclusion.
    
    **Parameters:**
    - **task_id**: Task identifier
    - **include_actions**: Include browser actions performed in each step
    - **include_screenshots**: Include screenshots taken during execution
    - **status_filter**: Filter steps by execution status
    
    **Returns:**
    - Ordered list of execution steps with optional related data
    """
    
    try:
        # Verify task exists and belongs to organization
        task_service = TaskService(db)
        task = await task_service.get_task_by_id(
            task_id=task_id,
            organization_id=organization.organization_id
        )
        
        if not task:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "TASK_NOT_FOUND",
                    "message": f"Task {task_id} not found"
                }
            )
        
        # Get steps with filtering
        step_service = StepService(db)
        steps = await step_service.get_steps_by_task(
            task_id=task_id,
            status_filter=status_filter,
            include_actions=include_actions,
            include_screenshots=include_screenshots
        )
        
        # Convert to response models
        step_responses = [
            ModelToSchemaConverter.step_to_response(
                step,
                include_actions=include_actions,
                include_screenshots=include_screenshots
            )
            for step in steps
        ]
        
        return JSONResponse(
            content={
                "success": True,
                "data": [step.dict() for step in step_responses],
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat(),
                    "total_steps": len(step_responses),
                    "filters_applied": {
                        "status_filter": status_filter,
                        "include_actions": include_actions,
                        "include_screenshots": include_screenshots
                    }
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "Get task steps failed",
            extra={
                "task_id": task_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "GET_STEPS_FAILED",
                "message": "Failed to retrieve task steps"
            }
        )

#### 6. Task Analytics Endpoint

```python
@router.get("/{task_id}/analytics", response_model=TaskAnalyticsResponse)
async def get_task_analytics(
    task_id: str,
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> TaskAnalyticsResponse:
    """
    Get comprehensive task analytics and performance metrics
    
    Provides detailed analytics about task execution including:
    - Performance metrics (timing, success rates)
    - Resource usage (screenshots, artifacts)
    - Error analysis and patterns
    - Execution efficiency metrics
    
    **Returns:**
    - Comprehensive analytics dashboard data
    """
    
    try:
        # Verify task access
        task_service = TaskService(db)
        task = await task_service.get_task_by_id(
            task_id=task_id,
            organization_id=organization.organization_id
        )
        
        if not task:
            raise HTTPException(status_code=404, detail="Task not found")
        
        # Get analytics data
        analytics_service = TaskAnalyticsService(db)
        analytics = await analytics_service.get_task_analytics(task_id)
        
        return JSONResponse(
            content={
                "success": True,
                "data": analytics.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except Exception as e:
        logger.error(f"Get task analytics failed: {e}")
        raise HTTPException(status_code=500, detail="Failed to retrieve analytics")

class TaskAnalyticsService:
    """Service for generating task analytics"""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def get_task_analytics(self, task_id: str) -> TaskAnalyticsData:
        """Generate comprehensive task analytics"""
        
        # Basic task metrics
        task_metrics = await self._get_task_metrics(task_id)
        
        # Step performance analysis
        step_metrics = await self._get_step_metrics(task_id)
        
        # Action analysis
        action_metrics = await self._get_action_metrics(task_id)
        
        # Resource usage
        resource_metrics = await self._get_resource_metrics(task_id)
        
        # Error analysis
        error_analysis = await self._get_error_analysis(task_id)
        
        return TaskAnalyticsData(
            task_id=task_id,
            task_metrics=task_metrics,
            step_metrics=step_metrics,
            action_metrics=action_metrics,
            resource_metrics=resource_metrics,
            error_analysis=error_analysis,
            generated_at=datetime.utcnow()
        )
    
    async def _get_task_metrics(self, task_id: str) -> Dict[str, Any]:
        """Get basic task execution metrics"""
        
        result = await self.db.execute(
            text("""
            SELECT 
                t.status,
                t.created_at,
                t.completed_at,
                EXTRACT(EPOCH FROM (t.completed_at - t.created_at)) as total_duration,
                COUNT(s.step_id) as total_steps,
                COUNT(CASE WHEN s.status = 'completed' THEN 1 END) as completed_steps,
                COUNT(CASE WHEN s.status = 'failed' THEN 1 END) as failed_steps,
                COUNT(a.action_id) as total_actions,
                COUNT(art.artifact_id) as total_artifacts
            FROM tasks t
            LEFT JOIN steps s ON t.task_id = s.task_id
            LEFT JOIN actions a ON s.step_id = a.step_id
            LEFT JOIN artifacts art ON t.task_id = art.task_id
            WHERE t.task_id = :task_id
            GROUP BY t.task_id, t.status, t.created_at, t.completed_at
            """),
            {"task_id": task_id}
        )
        
        row = result.fetchone()
        if not row:
            return {}
        
        success_rate = (row.completed_steps / row.total_steps * 100) if row.total_steps > 0 else 0
        
        return {
            "status": row.status,
            "total_duration_seconds": row.total_duration,
            "total_steps": row.total_steps,
            "completed_steps": row.completed_steps,
            "failed_steps": row.failed_steps,
            "success_rate": round(success_rate, 2),
            "total_actions": row.total_actions,
            "total_artifacts": row.total_artifacts,
            "actions_per_step": round(row.total_actions / row.total_steps, 2) if row.total_steps > 0 else 0
        }
```

---

## 📁 File 3: `skyvern/forge/api/endpoints/workflows.py` 🔥 **CRITICAL**

### Purpose
Workflow management endpoints for creating, executing, and monitoring complex automation workflows.

### Workflow API Architecture

```mermaid
graph TD
    A[Workflow API] --> B[Workflow Management]
    A --> C[Workflow Execution]
    A --> D[Workflow Monitoring]
    
    B --> E[Create Workflow]
    B --> F[Update Workflow]
    B --> G[List Workflows]
    B --> H[Get Workflow]
    B --> I[Delete Workflow]
    
    C --> J[Execute Workflow]
    C --> K[Get Workflow Run]
    C --> L[List Workflow Runs]
    C --> M[Cancel Workflow Run]
    
    D --> N[Workflow Analytics]
    D --> O[Performance Metrics]
    D --> P[Error Analysis]
    D --> Q[Resource Usage]
    
    E --> R[Schema Validation]
    E --> S[Block Validation]
    E --> T[Parameter Schema]
    
    J --> U[Parameter Validation]
    J --> V[Execution Queue]
    J --> W[State Management]
```

### Core Workflow Endpoints

#### 1. Create Workflow Endpoint

```python
from fastapi import APIRouter, Depends, HTTPException, Body
from typing import Dict, Any, List, Optional

router = APIRouter(prefix="/api/v1/workflows", tags=["workflows"])

@router.post("/", response_model=WorkflowResponse, status_code=201)
async def create_workflow(
    workflow_data: WorkflowCreate,
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> WorkflowResponse:
    """
    Create a new workflow template
    
    Creates a reusable workflow template that can be executed multiple times
    with different parameters. Workflows consist of connected blocks that
    define the automation logic.
    
    **Workflow Structure:**
    - **Blocks**: Individual automation components (navigation, extraction, etc.)
    - **Connections**: Define execution flow between blocks
    - **Parameters**: Input schema for workflow execution
    
    **Block Types:**
    - `navigation`: Navigate to URLs and interact with pages
    - `extraction`: Extract data from pages
    - `loop`: Iterate over data sets
    - `condition`: Conditional logic branches
    - `code`: Custom code execution
    
    **Returns:**
    - Created workflow with unique ID and validation status
    """
    
    try:
        # Validate workflow structure
        validation_result = await WorkflowValidator.validate_workflow(
            workflow_data, organization
        )
        
        if not validation_result.is_valid:
            raise HTTPException(
                status_code=400,
                detail={
                    "code": "WORKFLOW_VALIDATION_FAILED",
                    "message": "Workflow validation failed",
                    "errors": validation_result.errors,
                    "warnings": validation_result.warnings
                }
            )
        
        # Create workflow in database
        workflow_service = WorkflowService(db)
        workflow = await workflow_service.create_workflow(
            workflow_data=workflow_data,
            organization_id=organization.organization_id,
            created_by=context.user_id
        )
        
        # Convert to response model
        response = ModelToSchemaConverter.workflow_to_response(workflow)
        
        logger.info(
            "Workflow created successfully",
            extra={
                "workflow_id": workflow.workflow_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "block_count": len(workflow_data.workflow_definition.get("blocks", []))
            }
        )
        
        return JSONResponse(
            status_code=201,
            content={
                "success": True,
                "data": response.dict(),
                "validation": {
                    "is_valid": validation_result.is_valid,
                    "warnings": validation_result.warnings,
                    "block_count": len(workflow_data.workflow_definition.get("blocks", [])),
                    "parameter_count": len(workflow_data.parameter_schema.get("properties", {}))
                },
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except WorkflowValidationError as e:
        raise HTTPException(
            status_code=400,
            detail={
                "code": "VALIDATION_ERROR",
                "message": str(e),
                "validation_errors": e.validation_errors
            }
        )
    
    except Exception as e:
        logger.error(
            "Workflow creation failed",
            extra={
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "WORKFLOW_CREATION_FAILED",
                "message": "Failed to create workflow"
            }
        )

class WorkflowValidator:
    """Workflow validation logic"""
    
    @staticmethod
    async def validate_workflow(
        workflow_data: WorkflowCreate,
        organization: Organization
    ) -> WorkflowValidationResult:
        """Comprehensive workflow validation"""
        
        errors = []
        warnings = []
        
        # Basic validation
        if not workflow_data.title.strip():
            errors.append("Workflow title cannot be empty")
        
        if len(workflow_data.title) > 100:
            errors.append("Workflow title too long (max 100 characters)")
        
        # Workflow definition validation
        workflow_def = workflow_data.workflow_definition
        if not workflow_def:
            errors.append("Workflow definition is required")
            return WorkflowValidationResult(False, errors, warnings)
        
        # Validate blocks
        blocks = workflow_def.get("blocks", [])
        if not blocks:
            errors.append("Workflow must contain at least one block")
        
        # Validate each block
        block_ids = set()
        for i, block in enumerate(blocks):
            block_errors = await WorkflowValidator._validate_block(block, i)
            errors.extend(block_errors)
            
            # Check for duplicate block IDs
            block_id = block.get("id")
            if block_id in block_ids:
                errors.append(f"Duplicate block ID: {block_id}")
            else:
                block_ids.add(block_id)
        
        # Validate connections
        connections = workflow_def.get("connections", [])
        connection_errors = await WorkflowValidator._validate_connections(
            connections, block_ids
        )
        errors.extend(connection_errors)
        
        # Validate parameter schema
        if workflow_data.parameter_schema:
            param_errors = await WorkflowValidator._validate_parameter_schema(
                workflow_data.parameter_schema
            )
            errors.extend(param_errors)
        
        # Check organization limits
        if len(blocks) > organization.settings.get("max_workflow_blocks", 50):
            errors.append(f"Too many blocks (max: {organization.settings.get('max_workflow_blocks', 50)})")
        
        # Generate warnings for potential issues
        if len(blocks) > 20:
            warnings.append("Large workflow may have performance implications")
        
        if not workflow_data.description:
            warnings.append("Consider adding a description for better maintainability")
        
        return WorkflowValidationResult(
            is_valid=len(errors) == 0,
            errors=errors,
            warnings=warnings
        )
    
    @staticmethod
    async def _validate_block(block: Dict[str, Any], index: int) -> List[str]:
        """Validate individual workflow block"""
        
        errors = []
        
        # Required fields
        if "id" not in block:
            errors.append(f"Block {index}: Missing required field 'id'")
        
        if "type" not in block:
            errors.append(f"Block {index}: Missing required field 'type'")
        
        # Block type validation
        block_type = block.get("type")
        valid_types = ["navigation", "extraction", "loop", "condition", "code", "wait"]
        
        if block_type not in valid_types:
            errors.append(f"Block {index}: Invalid block type '{block_type}'")
        
        # Type-specific validation
        if block_type == "navigation":
            if "url" not in block.get("parameters", {}):
                errors.append(f"Navigation block {index}: Missing 'url' parameter")
        
        elif block_type == "extraction":
            if "data_extraction_goal" not in block.get("parameters", {}):
                errors.append(f"Extraction block {index}: Missing 'data_extraction_goal' parameter")
        
        elif block_type == "loop":
            loop_params = block.get("parameters", {})
            if "loop_over" not in loop_params:
                errors.append(f"Loop block {index}: Missing 'loop_over' parameter")
            if "loop_blocks" not in loop_params:
                errors.append(f"Loop block {index}: Missing 'loop_blocks' parameter")
        
        elif block_type == "condition":
            condition_params = block.get("parameters", {})
            if "condition_expression" not in condition_params:
                errors.append(f"Condition block {index}: Missing 'condition_expression' parameter")
        
        return errors
    
    @staticmethod
    async def _validate_connections(
        connections: List[Dict[str, Any]], 
        valid_block_ids: set
    ) -> List[str]:
        """Validate workflow connections"""
        
        errors = []
        
        for i, connection in enumerate(connections):
            # Required fields
            if "source" not in connection:
                errors.append(f"Connection {i}: Missing 'source' field")
                continue
            
            if "target" not in connection:
                errors.append(f"Connection {i}: Missing 'target' field")
                continue
            
            # Validate block IDs exist
            source_id = connection["source"]
            target_id = connection["target"]
            
            if source_id not in valid_block_ids:
                errors.append(f"Connection {i}: Invalid source block ID '{source_id}'")
            
            if target_id not in valid_block_ids:
                errors.append(f"Connection {i}: Invalid target block ID '{target_id}'")
            
            # Check for self-connections
            if source_id == target_id:
                errors.append(f"Connection {i}: Block cannot connect to itself")
        
        return errors

class WorkflowValidationResult(BaseModel):
    """Workflow validation result"""
    
    is_valid: bool
    errors: List[str]
    warnings: List[str]
```

#### 2. Execute Workflow Endpoint

```python
@router.post("/{workflow_id}/runs", response_model=WorkflowRunResponse, status_code=201)
async def execute_workflow(
    workflow_id: str,
    execution_data: WorkflowExecutionRequest,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> WorkflowRunResponse:
    """
    Execute a workflow with provided parameters
    
    Starts a new execution of the specified workflow template with the
    provided parameters. The workflow will be executed asynchronously.
    
    **Execution Process:**
    1. Validate workflow exists and is active
    2. Validate parameters against workflow schema
    3. Create workflow run record
    4. Queue for execution by workflow engine
    5. Return run ID for monitoring
    
    **Parameters:**
    - **workflow_id**: ID of workflow template to execute
    - **parameters**: Input parameters matching workflow schema
    - **priority**: Execution priority (normal, high, urgent)
    
    **Returns:**
    - Workflow run object with ID and initial status
    """
    
    try:
        # Get and validate workflow
        workflow_service = WorkflowService(db)
        workflow = await workflow_service.get_workflow_by_id(
            workflow_id=workflow_id,
            organization_id=organization.organization_id
        )
        
        if not workflow:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "WORKFLOW_NOT_FOUND",
                    "message": f"Workflow {workflow_id} not found"
                }
            )
        
        if not workflow.is_active:
            raise HTTPException(
                status_code=409,
                detail={
                    "code": "WORKFLOW_INACTIVE",
                    "message": "Cannot execute inactive workflow"
                }
            )
        
        # Validate parameters
        parameter_errors = await WorkflowValidator.validate_execution_parameters(
            execution_data.parameters,
            workflow.parameter_schema
        )
        
        if parameter_errors:
            raise HTTPException(
                status_code=400,
                detail={
                    "code": "PARAMETER_VALIDATION_FAILED",
                    "message": "Parameter validation failed",
                    "errors": parameter_errors
                }
            )
        
        # Check execution limits
        active_runs = await workflow_service.get_active_run_count(
            workflow_id=workflow_id,
            organization_id=organization.organization_id
        )
        
        max_concurrent = organization.settings.get("max_concurrent_workflow_runs", 5)
        if active_runs >= max_concurrent:
            raise HTTPException(
                status_code=429,
                detail={
                    "code": "EXECUTION_LIMIT_EXCEEDED",
                    "message": f"Maximum concurrent workflow runs exceeded ({max_concurrent})"
                }
            )
        
        # Create workflow run
        workflow_run = await workflow_service.create_workflow_run(
            workflow_id=workflow_id,
            organization_id=organization.organization_id,
            parameters=execution_data.parameters,
            priority=execution_data.priority,
            initiated_by=context.user_id
        )
        
        # Queue for execution
        background_tasks.add_task(
            queue_workflow_for_execution,
            workflow_run.workflow_run_id,
            execution_data.priority
        )
        
        # Convert to response
        response = ModelToSchemaConverter.workflow_run_to_response(workflow_run)
        
        logger.info(
            "Workflow execution started",
            extra={
                "workflow_id": workflow_id,
                "workflow_run_id": workflow_run.workflow_run_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "priority": execution_data.priority
            }
        )
        
        return JSONResponse(
            status_code=201,
            content={
                "success": True,
                "data": response.dict(),
                "execution_info": {
                    "priority": execution_data.priority,
                    "estimated_start_time": (
                        datetime.utcnow() + timedelta(seconds=30)
                    ).isoformat(),
                    "parameter_count": len(execution_data.parameters)
                },
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat()
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "Workflow execution failed",
            extra={
                "workflow_id": workflow_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "WORKFLOW_EXECUTION_FAILED",
                "message": "Failed to start workflow execution"
            }
        )

class WorkflowExecutionRequest(BaseModel):
    """Workflow execution request model"""
    
    parameters: Dict[str, Any] = Field(..., description="Workflow input parameters")
    priority: WorkflowPriority = Field(
        WorkflowPriority.normal, 
        description="Execution priority"
    )
    webhook_callback_url: Optional[str] = Field(
        None, 
        description="URL to call when workflow completes"
    )
    max_execution_time_minutes: Optional[int] = Field(
        None, 
        description="Maximum execution time before timeout"
    )
    
    @validator('max_execution_time_minutes')
    def validate_execution_time(cls, v):
        if v is not None and (v < 1 or v > 1440):  # 1 minute to 24 hours
            raise ValueError('Execution time must be between 1 and 1440 minutes')
        return v

class WorkflowPriority(str, Enum):
    """Workflow execution priority levels"""
    low = "low"
    normal = "normal"
    high = "high"
    urgent = "urgent"
```

#### 3. Get Workflow Run Endpoint

```python
@router.get("/{workflow_id}/runs/{run_id}", response_model=WorkflowRunResponse)
async def get_workflow_run(
    workflow_id: str,
    run_id: str,
    include_blocks: bool = Query(False, description="Include block execution details"),
    include_logs: bool = Query(False, description="Include execution logs"),
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> WorkflowRunResponse:
    """
    Get workflow run details and status
    
    Retrieves comprehensive information about a specific workflow execution,
    including current status, progress, and optional detailed execution data.
    
    **Information Included:**
    - Execution status and timing
    - Input parameters and outputs
    - Current block being executed
    - Error information if applicable
    - Optional: Block-by-block execution details
    - Optional: Detailed execution logs
    
    **Returns:**
    - Complete workflow run information with optional details
    """
    
    try:
        workflow_service = WorkflowService(db)
        
        # Verify workflow access
        workflow = await workflow_service.get_workflow_by_id(
            workflow_id=workflow_id,
            organization_id=organization.organization_id
        )
        
        if not workflow:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "WORKFLOW_NOT_FOUND",
                    "message": f"Workflow {workflow_id} not found"
                }
            )
        
        # Get workflow run
        workflow_run = await workflow_service.get_workflow_run_by_id(
            run_id=run_id,
            workflow_id=workflow_id,
            include_blocks=include_blocks,
            include_logs=include_logs
        )
        
        if not workflow_run: