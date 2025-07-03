# Phase 1: Foundation & Configuration
## Skyvern Repository Deep Dive

---

## 📋 Phase Overview

**Phase 1 Focus**: Understanding the core setup and configuration layer that powers the entire Skyvern application.

**Timeline**: Week 1 of the 12-phase learning plan

**Importance**: This is the foundation layer - everything else builds upon this configuration system.

---

## 🎯 Learning Goals for Phase 1

- ✅ **Configuration Loading**: Understand how Skyvern loads and validates settings
- ✅ **Environment Variables**: Know all required and optional environment variables  
- ✅ **Application Startup**: Understand the initialization sequence
- ✅ **Error Handling**: Grasp the exception hierarchy and error propagation
- ✅ **Constants Management**: Know how global constants are organized

---

## 🗂️ Critical Files to Study (Priority Order)

### 🔥 **CRITICAL** Files (Must Master)
1. **`skyvern/config.py`** - Central configuration management
2. **`skyvern/constants.py`** - Global constants and enums
3. **`skyvern/__init__.py`** - Application initialization
4. **`skyvern/exceptions.py`** - Exception hierarchy

### 🔶 **IMPORTANT** Files (Should Understand)
5. **`skyvern/__main__.py`** - Module execution entry point
6. **`skyvern/analytics.py`** - Telemetry and tracking
7. **`skyvern/_version.py`** - Version information

---

## 📁 File 1: `skyvern/config.py` 🔥 **CRITICAL**

### Purpose
Central configuration management using Pydantic Settings for type-safe, environment-aware configuration loading.

### Key Components

#### Configuration Class Structure
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=(".env", ".env.staging", ".env.prod"), 
        extra="ignore"
    )
    
    # Browser Configuration
    BROWSER_TYPE: str = "chromium-headful"
    BROWSER_REMOTE_DEBUGGING_URL: str = "http://127.0.0.1:9222"
    CHROME_EXECUTABLE_PATH: str | None = None
    
    # Database Configuration
    DATABASE_STRING: str = "postgresql+psycopg://skyvern@localhost/skyvern"
    DATABASE_STATEMENT_TIMEOUT_MS: int = 60000
    DISABLE_CONNECTION_POOL: bool = False
    
    # Performance & Limits
    MAX_STEPS_PER_RUN: int = 10
    MAX_STEPS_PER_TASK_V2: int = 25
    MAX_RETRIES_PER_STEP: int = 5
    BROWSER_ACTION_TIMEOUT_MS: int = 5000
```

#### Configuration Categories

| Category | Settings Count | Purpose | Critical Settings |
|----------|----------------|---------|-------------------|
| **Browser** | 8 settings | Browser automation setup | `BROWSER_TYPE`, `CHROME_EXECUTABLE_PATH` |
| **Database** | 3 settings | Database connectivity | `DATABASE_STRING`, `DATABASE_STATEMENT_TIMEOUT_MS` |
| **Performance** | 12 settings | Timeouts and limits | `MAX_STEPS_PER_RUN`, `BROWSER_ACTION_TIMEOUT_MS` |
| **Security** | 6 settings | API security | `ALLOWED_ORIGINS`, `BLOCKED_HOSTS` |
| **Logging** | 5 settings | Debug and monitoring | `LOG_LEVEL`, `JSON_LOGGING` |
| **Infrastructure** | 4 settings | Deployment config | `ENV`, `PORT` |

#### Environment Variable Loading Flow

```mermaid
flowchart TD
    A[Application Start] --> B[Pydantic Settings Init]
    B --> C[Load .env files]
    C --> D{Environment Variables?}
    D -->|Yes| E[Override defaults]
    D -->|No| F[Use defaults]
    E --> G[Type Validation]
    F --> G
    G --> H{Validation Success?}
    H -->|Yes| I[Settings Object Created]
    H -->|No| J[ValidationError]
    I --> K[Application Ready]
    J --> L[Application Fails to Start]
```

#### Critical Environment Variables

| Variable | Type | Default | Purpose | Required |
|----------|------|---------|---------|----------|
| `DATABASE_STRING` | str | Local PostgreSQL | Database connection | ✅ |
| `BROWSER_TYPE` | str | chromium-headful | Browser automation type | ❌ |
| `MAX_STEPS_PER_RUN` | int | 10 | Task execution limit | ❌ |
| `LOG_LEVEL` | str | INFO | Logging verbosity | ❌ |
| `ENV` | str | local | Environment identifier | ❌ |
| `PORT` | int | 8000 | API server port | ❌ |

#### Browser Configuration Options

```python
# Browser Types Available
BROWSER_TYPES = [
    "chromium-headful",    # Chrome with UI (development)
    "chromium-headless",   # Chrome without UI (production)
    "firefox-headful",     # Firefox with UI
    "firefox-headless",    # Firefox without UI
    "webkit-headful",      # Safari-based (macOS only)
    "webkit-headless"      # Safari-based headless
]

# Browser-Related Settings
BROWSER_REMOTE_DEBUGGING_URL = "http://127.0.0.1:9222"  # Chrome DevTools
CHROME_EXECUTABLE_PATH = None  # Auto-detect or specify path
VIDEO_PATH = "./video"         # Screen recording storage
HAR_PATH = "./har"            # Network activity logs
```

#### Performance Tuning Settings

| Setting | Default | Purpose | Tuning Guidelines |
|---------|---------|---------|-------------------|
| `BROWSER_ACTION_TIMEOUT_MS` | 5000 | Individual action timeout | Increase for slow sites |
| `BROWSER_LOADING_TIMEOUT_MS` | 90000 | Page load timeout | Adjust based on target sites |
| `MAX_STEPS_PER_RUN` | 10 | Steps before task completion | Balance automation vs. infinite loops |
| `MAX_RETRIES_PER_STEP` | 5 | Retry attempts per failed step | Higher for unreliable sites |

### Configuration Best Practices

1. **Environment Separation**
   ```bash
   # Development
   .env              # Local development settings
   
   # Staging  
   .env.staging      # Staging environment settings
   
   # Production
   .env.prod         # Production environment settings
   ```

2. **Security Considerations**
   - Never commit API keys or secrets to version control
   - Use environment variables for sensitive data
   - Validate all input settings with Pydantic

3. **Performance Optimization**
   - Tune timeouts based on target websites
   - Adjust step limits to prevent infinite loops
   - Configure appropriate database timeouts

---

## 📁 File 2: `skyvern/constants.py` 🔥 **CRITICAL**

### Purpose
Centralized storage for global constants, enums, and configuration values used throughout the application.

### Key Constants Categories

#### File System Paths
```python
import os
from pathlib import Path

# Base directory for Skyvern
SKYVERN_DIR = Path(__file__).parent.parent.absolute()

# Default storage paths
DEFAULT_VIDEO_PATH = "./video"
DEFAULT_HAR_PATH = "./har" 
DEFAULT_LOG_PATH = "./log"
DEFAULT_TEMP_PATH = "./temp"
DEFAULT_DOWNLOADS_PATH = "./downloads"

# Artifact storage
ARTIFACT_STORAGE_PATH = os.getenv("ARTIFACT_STORAGE_PATH", "./artifacts")
SCREENSHOT_STORAGE_PATH = os.getenv("SCREENSHOT_STORAGE_PATH", "./screenshots")
```

#### Timeout Constants
```python
# Browser interaction timeouts (milliseconds)
DEFAULT_TIMEOUT_MS = 5000
SHORT_TIMEOUT_MS = 2000
MEDIUM_TIMEOUT_MS = 10000
LONG_TIMEOUT_MS = 30000
EXTRA_LONG_TIMEOUT_MS = 60000

# Network timeouts
HTTP_REQUEST_TIMEOUT_SEC = 30
DOWNLOAD_TIMEOUT_SEC = 300
UPLOAD_TIMEOUT_SEC = 600

# Database timeouts
DB_QUERY_TIMEOUT_SEC = 30
DB_CONNECTION_TIMEOUT_SEC = 10
```

#### Status Enumerations
```python
# Task execution statuses
class TaskStatus(str, Enum):
    CREATED = "created"
    QUEUED = "queued" 
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"
    TERMINATED = "terminated"

# Step execution statuses  
class StepStatus(str, Enum):
    CREATED = "created"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    RETRYING = "retrying"
    SKIPPED = "skipped"

# Workflow statuses
class WorkflowStatus(str, Enum):
    CREATED = "created"
    RUNNING = "running" 
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"
```

#### Action Type Constants
```python
# Browser action types
class ActionType(str, Enum):
    CLICK = "click"
    TYPE = "type"
    SELECT = "select"
    SCROLL = "scroll"
    NAVIGATE = "navigate"
    WAIT = "wait"
    EXTRACT = "extract"
    SCREENSHOT = "screenshot"
    UPLOAD_FILE = "upload_file"
    DOWNLOAD_FILE = "download_file"
    COMPLETE = "complete"
    TERMINATE = "terminate"

# Element interaction types
class ElementType(str, Enum):
    BUTTON = "button"
    INPUT = "input"
    SELECT = "select"
    LINK = "link"
    CHECKBOX = "checkbox"
    RADIO = "radio"
    TEXTAREA = "textarea"
    IMAGE = "image"
    TABLE = "table"
```

#### Browser Configuration Constants
```python
# Supported browsers
SUPPORTED_BROWSERS = ["chromium", "firefox", "webkit"]

# Browser launch arguments
CHROMIUM_DEFAULT_ARGS = [
    "--no-sandbox",
    "--disable-blink-features=AutomationControlled",
    "--disable-extensions-except",
    "--disable-extensions",
    "--disable-plugins-discovery",
    "--disable-dev-shm-usage"
]

FIREFOX_DEFAULT_ARGS = [
    "--no-remote",
    "--new-instance"  
]

# Viewport sizes
VIEWPORT_SIZES = {
    "desktop": (1920, 1080),
    "laptop": (1366, 768), 
    "tablet": (768, 1024),
    "mobile": (375, 667)
}
```

#### API Configuration Constants
```python
# API versioning
API_VERSION_V1 = "v1"
API_VERSION_V2 = "v2"
CURRENT_API_VERSION = API_VERSION_V2

# HTTP status codes
HTTP_STATUS_CODES = {
    "SUCCESS": 200,
    "CREATED": 201,
    "BAD_REQUEST": 400,
    "UNAUTHORIZED": 401,
    "FORBIDDEN": 403,
    "NOT_FOUND": 404,
    "INTERNAL_ERROR": 500
}

# Rate limiting
DEFAULT_RATE_LIMIT_PER_MINUTE = 100
DEFAULT_RATE_LIMIT_PER_HOUR = 1000
```

### Constants Usage Patterns

#### Importing Constants
```python
# Import specific constants
from skyvern.constants import TaskStatus, ActionType, DEFAULT_TIMEOUT_MS

# Import all constants (not recommended)
from skyvern.constants import *

# Import with alias
from skyvern.constants import SKYVERN_DIR as BASE_DIR
```

#### Using Enums in Code
```python
# Correct usage
if task.status == TaskStatus.COMPLETED:
    print("Task finished successfully")

# Type checking with enums
def process_task(task_status: TaskStatus) -> bool:
    return task_status in [TaskStatus.COMPLETED, TaskStatus.FAILED]
```

---

## 📁 File 3: `skyvern/__init__.py` 🔥 **CRITICAL**

### Purpose
Main application initialization and component setup. This file controls the startup sequence and global imports.

### Initialization Sequence

```python
"""
Skyvern - AI-powered web automation platform
"""

# Version information
from skyvern._version import __version__

# Core configuration
from skyvern.config import Settings
from skyvern.constants import *

# Exception handling
from skyvern.exceptions import *

# Initialize global settings
settings = Settings()

# Setup logging
import logging
from skyvern.utils.logging import setup_logging

setup_logging(
    level=settings.LOG_LEVEL,
    json_format=settings.JSON_LOGGING,
    log_file=settings.LOG_PATH
)

logger = logging.getLogger(__name__)
logger.info(f"Skyvern v{__version__} initializing...")

# Initialize core components
def initialize_application():
    """Initialize all application components in correct order"""
    
    # 1. Database connection
    from skyvern.forge.sdk.db.client import initialize_database
    initialize_database(settings.DATABASE_STRING)
    
    # 2. Browser manager
    from skyvern.webeye.browser_manager import BrowserManager
    browser_manager = BrowserManager(settings)
    
    # 3. AI/LLM clients
    from skyvern.forge.sdk.api.llm import initialize_llm_clients
    initialize_llm_clients(settings)
    
    # 4. Analytics (if enabled)
    if settings.ANALYTICS_ENABLED:
        from skyvern.analytics import initialize_analytics
        initialize_analytics(settings)
    
    logger.info("Skyvern initialization complete")
    
    return {
        "browser_manager": browser_manager,
        "settings": settings,
        "version": __version__
    }

# Global application state
_app_state = None

def get_app_state():
    """Get or initialize application state"""
    global _app_state
    if _app_state is None:
        _app_state = initialize_application()
    return _app_state

# Public API exports
__all__ = [
    "__version__",
    "settings", 
    "initialize_application",
    "get_app_state",
    "TaskStatus",
    "StepStatus", 
    "ActionType",
    "SkyvernException"
]
```

### Component Dependencies

```mermaid
graph TD
    A[__init__.py] --> B[config.py]
    A --> C[constants.py]
    A --> D[exceptions.py]
    A --> E[_version.py]
    B --> F[Environment Variables]
    B --> G[Pydantic Validation]
    D --> H[Custom Exception Classes]
    A --> I[Database Client]
    A --> J[Browser Manager]
    A --> K[LLM Clients]
    A --> L[Analytics]
```

### Startup Validation

```python
def validate_startup_requirements():
    """Validate all requirements before application starts"""
    
    validation_results = {
        "config": False,
        "database": False, 
        "browser": False,
        "llm": False
    }
    
    # Validate configuration
    try:
        settings = Settings()
        validation_results["config"] = True
    except Exception as e:
        logger.error(f"Configuration validation failed: {e}")
        
    # Validate database connection
    try:
        from skyvern.forge.sdk.db.client import test_database_connection
        test_database_connection(settings.DATABASE_STRING)
        validation_results["database"] = True
    except Exception as e:
        logger.error(f"Database validation failed: {e}")
        
    # Validate browser availability
    try:
        from skyvern.webeye.browser_factory import test_browser_availability
        test_browser_availability(settings.BROWSER_TYPE)
        validation_results["browser"] = True
    except Exception as e:
        logger.error(f"Browser validation failed: {e}")
        
    # Validate LLM configuration
    try:
        from skyvern.forge.sdk.api.llm import test_llm_connection
        test_llm_connection(settings)
        validation_results["llm"] = True
    except Exception as e:
        logger.error(f"LLM validation failed: {e}")
        
    return validation_results
```

---

## 📁 File 4: `skyvern/exceptions.py` 🔥 **IMPORTANT**

### Purpose
Comprehensive exception hierarchy for structured error handling throughout the application.

### Exception Hierarchy

```python
class SkyvernException(Exception):
    """Base exception for all Skyvern-related errors"""
    
    def __init__(self, message: str, error_code: str = None, details: Dict = None):
        self.message = message
        self.error_code = error_code or self.__class__.__name__
        self.details = details or {}
        self.timestamp = datetime.utcnow()
        super().__init__(self.message)
    
    def to_dict(self) -> Dict[str, Any]:
        return {
            "error_code": self.error_code,
            "message": self.message,
            "details": self.details,
            "timestamp": self.timestamp.isoformat(),
            "exception_type": self.__class__.__name__
        }

# Configuration Errors
class ConfigurationError(SkyvernException):
    """Raised when configuration is invalid or missing"""
    pass

class EnvironmentVariableError(ConfigurationError):
    """Raised when required environment variable is missing"""
    pass

# Database Errors  
class DatabaseError(SkyvernException):
    """Base class for database-related errors"""
    pass

class DatabaseConnectionError(DatabaseError):
    """Raised when database connection fails"""
    pass

class DatabaseQueryError(DatabaseError):
    """Raised when database query fails"""
    pass

# Browser Errors
class BrowserError(SkyvernException):
    """Base class for browser-related errors"""
    pass

class BrowserLaunchError(BrowserError):
    """Raised when browser fails to launch"""
    pass

class PageLoadError(BrowserError):
    """Raised when page fails to load"""
    pass

class ElementNotFoundError(BrowserError):
    """Raised when expected element is not found"""
    pass

class ActionTimeoutError(BrowserError):
    """Raised when browser action times out"""
    pass

# AI/LLM Errors
class AIError(SkyvernException):
    """Base class for AI-related errors"""
    pass

class LLMResponseError(AIError):
    """Raised when LLM response is invalid"""
    pass

class PromptValidationError(AIError):
    """Raised when prompt validation fails"""
    pass

# Workflow Errors
class WorkflowError(SkyvernException):
    """Base class for workflow-related errors"""
    pass

class StepExecutionError(WorkflowError):
    """Raised when workflow step execution fails"""
    pass

class WorkflowValidationError(WorkflowError):
    """Raised when workflow validation fails"""
    pass
```

### Error Handling Patterns

#### Exception Context Manager
```python
from contextlib import contextmanager

@contextmanager
def handle_skyvern_errors():
    """Context manager for structured error handling"""
    try:
        yield
    except SkyvernException:
        # Re-raise Skyvern exceptions as-is
        raise
    except Exception as e:
        # Wrap unknown exceptions
        raise SkyvernException(
            message=f"Unexpected error: {str(e)}",
            error_code="UNKNOWN_ERROR",
            details={"original_exception": type(e).__name__}
        )
```

#### Error Logging
```python
import logging

logger = logging.getLogger(__name__)

def log_exception(exc: SkyvernException, context: Dict = None):
    """Log exception with structured data"""
    
    log_data = exc.to_dict()
    if context:
        log_data["context"] = context
        
    if isinstance(exc, (DatabaseError, BrowserLaunchError)):
        logger.error("Critical error occurred", extra=log_data)
    elif isinstance(exc, (ActionTimeoutError, ElementNotFoundError)):
        logger.warning("Recoverable error occurred", extra=log_data)
    else:
        logger.info("Minor error occurred", extra=log_data)
```

#### HTTP Error Mapping
```python
def map_exception_to_http_status(exc: SkyvernException) -> int:
    """Map exceptions to appropriate HTTP status codes"""
    
    error_mappings = {
        ConfigurationError: 500,  # Internal Server Error
        DatabaseError: 500,       # Internal Server Error  
        BrowserError: 503,        # Service Unavailable
        AIError: 502,            # Bad Gateway
        WorkflowValidationError: 400,  # Bad Request
        ElementNotFoundError: 404,     # Not Found
    }
    
    for error_type, status_code in error_mappings.items():
        if isinstance(exc, error_type):
            return status_code
            
    return 500  # Default to Internal Server Error
```

---

## 📊 Configuration Data Flow

### Environment Loading Process

```mermaid
sequenceDiagram
    participant App as Application
    participant Env as Environment
    participant Pydantic as Pydantic Settings
    participant Validation as Validation Engine
    participant Config as Config Object
    
    App->>Pydantic: Initialize Settings()
    Pydantic->>Env: Read .env files
    Env-->>Pydantic: Environment variables
    Pydantic->>Env: Read system environment
    Env-->>Pydantic: System variables
    Pydantic->>Validation: Validate types & constraints
    Validation-->>Pydantic: Validation results
    Pydantic->>Config: Create Settings object
    Config-->>App: Ready for use
```

### Configuration Priority Order

| Priority | Source | Example | Override Behavior |
|----------|--------|---------|-------------------|
| **1 (Highest)** | System Environment Variables | `export DATABASE_STRING="..."` | Overrides everything |
| **2** | .env.prod file | `DATABASE_STRING=...` | Overrides lower priority |
| **3** | .env.staging file | `DATABASE_STRING=...` | Overrides lower priority |
| **4** | .env file | `DATABASE_STRING=...` | Overrides defaults only |
| **5 (Lowest)** | Code Defaults | `DATABASE_STRING: str = "..."` | Used when nothing else set |

---

## 🔧 Practical Examples

### Example 1: Local Development Setup

```bash
# .env file for local development
ENV=local
DEBUG_MODE=true
LOG_LEVEL=DEBUG
DATABASE_STRING=postgresql+psycopg://skyvern@localhost/skyvern_dev
BROWSER_TYPE=chromium-headful
MAX_STEPS_PER_RUN=5
BROWSER_ACTION_TIMEOUT_MS=10000
```

### Example 2: Production Configuration

```bash
# .env.prod file for production
ENV=production  
DEBUG_MODE=false
LOG_LEVEL=INFO
JSON_LOGGING=true
DATABASE_STRING=postgresql+psycopg://user:pass@prod-db:5432/skyvern
BROWSER_TYPE=chromium-headless
MAX_STEPS_PER_RUN=25
BROWSER_ACTION_TIMEOUT_MS=5000
ALLOWED_ORIGINS=["https://app.skyvern.com"]
```

### Example 3: Docker Configuration

```yaml
# docker-compose.yml
version: '3.8'
services:
  skyvern:
    build: .
    environment:
      - DATABASE_STRING=postgresql+psycopg://skyvern:password@postgres:5432/skyvern
      - BROWSER_TYPE=chromium-headless
      - ENV=docker
      - LOG_LEVEL=INFO
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=skyvern
      - POSTGRES_USER=skyvern
      - POSTGRES_PASSWORD=password
```

---

## ✅ Phase 1 Completion Checklist

### Understanding Verification

- [ ] **Configuration System**: Can explain how Pydantic Settings works
- [ ] **Environment Variables**: Know all critical environment variables and their purposes
- [ ] **Application Startup**: Understand the initialization sequence in `__init__.py`
- [ ] **Constants Organization**: Know how constants are categorized and used
- [ ] **Exception Handling**: Understand the exception hierarchy and error handling patterns
- [ ] **File Dependencies**: Can map the dependency relationships between foundation files

### Practical Skills

- [ ] **Environment Setup**: Can configure Skyvern for different environments
- [ ] **Troubleshooting**: Can debug configuration-related startup issues
- [ ] **Customization**: Can modify settings for specific use cases
- [ ] **Error Handling**: Can implement proper exception handling in new code

### Key Questions to Answer

1. **What happens during application startup?**
   - Pydantic loads settings from environment and .env files
   - Settings are validated for type correctness and constraints
   - Core components are initialized in dependency order
   - Global application state is established

2. **How are environment variables prioritized?**
   - System environment variables (highest priority)
   - .env.prod file
   - .env.staging file  
   - .env file
   - Code defaults (lowest priority)

3. **What are the critical configuration categories?**
   - Browser automation settings
   - Database connectivity
   - Performance and timeout settings
   - Security and access control
   - Logging and debugging

4. **How does error handling work?**
   - Hierarchical exception system with SkyvernException base
   - Structured error data with timestamps and context
   - HTTP status code mapping for API responses
   - Comprehensive logging with error categorization

---

## 🚀 Ready for Phase 2?

You should now have a solid understanding of Skyvern's foundation layer. The configuration system, constants management, initialization sequence, and error handling patterns form the bedrock that all other components build upon.

**Next Phase Preview**: Phase 2 will dive into the Database Layer, where you'll learn about:
- Database models and relationships
- SQLAlchemy integration patterns
- Data schemas and validation
- CRUD operations and query optimization

Type "ready for phase 2" when you want to continue with the Database Layer deep dive!