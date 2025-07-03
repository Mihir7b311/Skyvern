        if not workflow_run:
            raise HTTPException(
                status_code=404,
                detail={
                    "code": "WORKFLOW_RUN_NOT_FOUND",
                    "message": f"Workflow run {run_id} not found"
                }
            )
        
        # Convert to response model
        response = ModelToSchemaConverter.workflow_run_to_response(
            workflow_run,
            include_blocks=include_blocks,
            include_logs=include_logs
        )
        
        # Add execution progress information
        if workflow_run.status == WorkflowRunStatus.running:
            progress_info = await workflow_service.get_execution_progress(run_id)
            response.execution_progress = progress_info
        
        return JSONResponse(
            content={
                "success": True,
                "data": response.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat(),
                    "includes": {
                        "blocks": include_blocks,
                        "logs": include_logs
                    }
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "Get workflow run failed",
            extra={
                "workflow_id": workflow_id,
                "run_id": run_id,
                "organization_id": organization.organization_id,
                "request_id": context.request_id,
                "error": str(e)
            }
        )
        raise HTTPException(
            status_code=500,
            detail={
                "code": "GET_WORKFLOW_RUN_FAILED",
                "message": "Failed to retrieve workflow run"
            }
        )

#### 4. Workflow Analytics Endpoint

```python
@router.get("/{workflow_id}/analytics", response_model=WorkflowAnalyticsResponse)
async def get_workflow_analytics(
    workflow_id: str,
    days: int = Query(30, ge=1, le=90, description="Analysis period in days"),
    include_performance: bool = Query(True, description="Include performance metrics"),
    include_errors: bool = Query(True, description="Include error analysis"),
    include_usage: bool = Query(True, description="Include usage patterns"),
    db: AsyncSession = Depends(Dependencies.get_database),
    organization: Organization = Depends(Dependencies.get_current_organization),
    context: RequestContext = Depends(Dependencies.get_request_context)
) -> WorkflowAnalyticsResponse:
    """
    Get comprehensive workflow analytics
    
    Provides detailed analytics and insights about workflow performance,
    usage patterns, error rates, and optimization opportunities.
    
    **Analytics Categories:**
    - **Performance**: Execution times, success rates, bottleneck analysis
    - **Usage**: Execution frequency, parameter patterns, user adoption
    - **Errors**: Failure analysis, common error patterns, troubleshooting
    - **Resources**: CPU/memory usage, artifact generation, cost analysis
    
    **Time Periods:**
    - Last 7 days: Recent performance trends
    - Last 30 days: Monthly patterns and reliability
    - Last 90 days: Long-term trends and seasonal patterns
    
    **Returns:**
    - Comprehensive analytics dashboard with actionable insights
    """
    
    try:
        # Verify workflow access
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
        
        # Generate analytics
        analytics_service = WorkflowAnalyticsService(db)
        analytics = await analytics_service.generate_workflow_analytics(
            workflow_id=workflow_id,
            days=days,
            include_performance=include_performance,
            include_errors=include_errors,
            include_usage=include_usage
        )
        
        return JSONResponse(
            content={
                "success": True,
                "data": analytics.dict(),
                "metadata": {
                    "request_id": context.request_id,
                    "timestamp": datetime.utcnow().isoformat(),
                    "analysis_period_days": days,
                    "includes": {
                        "performance": include_performance,
                        "errors": include_errors,
                        "usage": include_usage
                    }
                }
            }
        )
        
    except Exception as e:
        logger.error(
            "Get workflow analytics failed",
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
                "code": "GET_WORKFLOW_ANALYTICS_FAILED",
                "message": "Failed to retrieve workflow analytics"
            }
        )

class WorkflowAnalyticsService:
    """Service for generating workflow analytics"""
    
    def __init__(self, db: AsyncSession):
        self.db = db
    
    async def generate_workflow_analytics(
        self,
        workflow_id: str,
        days: int,
        include_performance: bool = True,
        include_errors: bool = True,
        include_usage: bool = True
    ) -> WorkflowAnalyticsData:
        """Generate comprehensive workflow analytics"""
        
        cutoff_date = datetime.utcnow() - timedelta(days=days)
        
        analytics_data = WorkflowAnalyticsData(
            workflow_id=workflow_id,
            period_days=days,
            generated_at=datetime.utcnow()
        )
        
        # Performance metrics
        if include_performance:
            analytics_data.performance_metrics = await self._get_performance_metrics(
                workflow_id, cutoff_date
            )
        
        # Error analysis
        if include_errors:
            analytics_data.error_analysis = await self._get_error_analysis(
                workflow_id, cutoff_date
            )
        
        # Usage patterns
        if include_usage:
            analytics_data.usage_patterns = await self._get_usage_patterns(
                workflow_id, cutoff_date
            )
        
        # Block performance analysis
        analytics_data.block_performance = await self._get_block_performance(
            workflow_id, cutoff_date
        )
        
        # Optimization recommendations
        analytics_data.recommendations = await self._generate_recommendations(
            analytics_data
        )
        
        return analytics_data
    
    async def _get_performance_metrics(
        self, 
        workflow_id: str, 
        cutoff_date: datetime
    ) -> Dict[str, Any]:
        """Get workflow performance metrics"""
        
        result = await self.db.execute(
            text("""
            WITH daily_stats AS (
                SELECT 
                    DATE_TRUNC('day', created_at) as date,
                    COUNT(*) as total_runs,
                    COUNT(CASE WHEN status = 'completed' THEN 1 END) as successful_runs,
                    COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_runs,
                    AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration,
                    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))) as median_duration,
                    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (completed_at - started_at))) as p95_duration
                FROM workflow_runs 
                WHERE workflow_id = :workflow_id 
                AND created_at >= :cutoff_date
                AND completed_at IS NOT NULL
                GROUP BY DATE_TRUNC('day', created_at)
                ORDER BY date
            ),
            overall_stats AS (
                SELECT 
                    COUNT(*) as total_runs,
                    COUNT(CASE WHEN status = 'completed' THEN 1 END) as successful_runs,
                    COUNT(CASE WHEN status = 'failed' THEN 1 END) as failed_runs,
                    AVG(EXTRACT(EPOCH FROM (completed_at - started_at))) as avg_duration,
                    MIN(EXTRACT(EPOCH FROM (completed_at - started_at))) as min_duration,
                    MAX(EXTRACT(EPOCH FROM (completed_at - started_at))) as max_duration
                FROM workflow_runs 
                WHERE workflow_id = :workflow_id 
                AND created_at >= :cutoff_date
                AND completed_at IS NOT NULL
            )
            SELECT 
                (SELECT json_agg(row_to_json(ds)) FROM daily_stats ds) as daily_stats,
                (SELECT row_to_json(os) FROM overall_stats os) as overall_stats
            """),
            {"workflow_id": workflow_id, "cutoff_date": cutoff_date}
        )
        
        row = result.fetchone()
        
        overall = row.overall_stats or {}
        success_rate = (
            (overall.get("successful_runs", 0) / overall.get("total_runs", 1)) * 100
            if overall.get("total_runs", 0) > 0 else 0
        )
        
        return {
            "total_runs": overall.get("total_runs", 0),
            "successful_runs": overall.get("successful_runs", 0),
            "failed_runs": overall.get("failed_runs", 0),
            "success_rate": round(success_rate, 2),
            "avg_duration_seconds": overall.get("avg_duration"),
            "min_duration_seconds": overall.get("min_duration"),
            "max_duration_seconds": overall.get("max_duration"),
            "daily_trends": row.daily_stats or []
        }
    
    async def _get_error_analysis(
        self, 
        workflow_id: str, 
        cutoff_date: datetime
    ) -> Dict[str, Any]:
        """Analyze workflow errors and failure patterns"""
        
        result = await self.db.execute(
            text("""
            WITH error_breakdown AS (
                SELECT 
                    failure_reason,
                    COUNT(*) as error_count,
                    COUNT(*) * 100.0 / (
                        SELECT COUNT(*) FROM workflow_runs 
                        WHERE workflow_id = :workflow_id 
                        AND status = 'failed'
                        AND created_at >= :cutoff_date
                    ) as error_percentage,
                    AVG(current_block_index) as avg_failure_block
                FROM workflow_runs 
                WHERE workflow_id = :workflow_id 
                AND status = 'failed'
                AND created_at >= :cutoff_date
                AND failure_reason IS NOT NULL
                GROUP BY failure_reason
                ORDER BY error_count DESC
                LIMIT 10
            ),
            block_failures AS (
                SELECT 
                    wb.block_type,
                    wb.block_index,
                    COUNT(*) as failure_count
                FROM workflow_blocks wb
                JOIN workflow_runs wr ON wb.workflow_run_id = wr.workflow_run_id
                WHERE wr.workflow_id = :workflow_id
                AND wb.status = 'failed'
                AND wr.created_at >= :cutoff_date
                GROUP BY wb.block_type, wb.block_index
                ORDER BY failure_count DESC
                LIMIT 10
            )
            SELECT 
                (SELECT json_agg(row_to_json(eb)) FROM error_breakdown eb) as error_breakdown,
                (SELECT json_agg(row_to_json(bf)) FROM block_failures bf) as block_failures
            """),
            {"workflow_id": workflow_id, "cutoff_date": cutoff_date}
        )
        
        row = result.fetchone()
        
        return {
            "error_breakdown": row.error_breakdown or [],
            "block_failures": row.block_failures or [],
            "total_errors": sum(
                error.get("error_count", 0) 
                for error in (row.error_breakdown or [])
            )
        }
```

---

## 📁 File 4: `skyvern/forge/sdk/services/org_auth_service.py` 🔥 **CRITICAL**

### Purpose
Organization-based authentication service handling API key validation and access control.

### Authentication Architecture

```mermaid
graph TD
    A[API Request] --> B[Extract API Key]
    B --> C[API Key Validation]
    C --> D[Organization Lookup]
    D --> E[Permission Check]
    E --> F[Rate Limit Check]
    F --> G[Request Processing]
    
    C --> H[Invalid Key]
    D --> I[Organization Not Found]
    E --> J[Access Denied]
    F --> K[Rate Limited]
    
    H --> L[401 Unauthorized]
    I --> L
    J --> M[403 Forbidden]
    K --> N[429 Too Many Requests]
    
    G --> O[Success Response]
```

### Authentication Service Implementation

```python
from typing import Optional, Dict, Any, List
from datetime import datetime, timedelta
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC
import secrets
import base64
import redis
import json

class OrganizationAuthService:
    """Organization authentication and authorization service"""
    
    def __init__(self, db: AsyncSession, redis_client: Optional[redis.Redis] = None):
        self.db = db
        self.redis = redis_client
        self.api_key_cache_ttl = 300  # 5 minutes
        self.rate_limit_window = 3600  # 1 hour
    
    async def authenticate_api_key(self, api_key: str) -> Optional[Organization]:
        """
        Authenticate API key and return associated organization
        
        **Process:**
        1. Validate API key format
        2. Check cache for recent validation
        3. Query database for API key
        4. Validate organization status
        5. Cache result for performance
        
        **Returns:**
        - Organization object if valid
        - None if invalid or expired
        """
        
        if not self._validate_api_key_format(api_key):
            return None
        
        # Check cache first
        cached_org = await self._get_cached_organization(api_key)
        if cached_org:
            return cached_org
        
        try:
            # Hash API key for database lookup
            api_key_hash = self._hash_api_key(api_key)
            
            # Query database
            result = await self.db.execute(
                text("""
                SELECT 
                    o.organization_id,
                    o.organization_name,
                    o.domain,
                    o.is_active,
                    o.settings,
                    o.max_steps_per_run,
                    o.max_retries_per_step,
                    ak.api_key_id,
                    ak.key_name,
                    ak.permissions,
                    ak.is_active as key_active,
                    ak.expires_at,
                    ak.last_used_at,
                    ak.usage_count
                FROM organizations o
                JOIN api_keys ak ON o.organization_id = ak.organization_id
                WHERE ak.key_hash = :api_key_hash
                AND o.is_active = true
                AND ak.is_active = true
                AND (ak.expires_at IS NULL OR ak.expires_at > now())
                """),
                {"api_key_hash": api_key_hash}
            )
            
            row = result.fetchone()
            if not row:
                return None
            
            # Create organization object
            organization = Organization(
                organization_id=row.organization_id,
                organization_name=row.organization_name,
                domain=row.domain,
                is_active=row.is_active,
                settings=row.settings or {},
                max_steps_per_run=row.max_steps_per_run,
                max_retries_per_step=row.max_retries_per_step
            )
            
            # Update API key usage
            await self._update_api_key_usage(row.api_key_id)
            
            # Cache organization
            await self._cache_organization(api_key, organization)
            
            return organization
            
        except Exception as e:
            logger.error(f"API key authentication failed: {e}")
            return None
    
    async def check_permissions(
        self, 
        organization: Organization, 
        resource: str, 
        action: str
    ) -> bool:
        """
        Check if organization has permission for specific resource/action
        
        **Permission Structure:**
        - Resources: tasks, workflows, browser_sessions, artifacts, analytics
        - Actions: create, read, update, delete, execute, cancel
        
        **Permission Levels:**
        - admin: All permissions
        - read_write: Create, read, update, execute
        - read_only: Read permissions only
        - custom: Specific permission list
        """
        
        try:
            # Get API key permissions from current context
            permissions = getattr(organization, 'api_key_permissions', {})
            
            if not permissions:
                # Default permissions for organizations
                permissions = organization.settings.get('default_permissions', {
                    'tasks': ['create', 'read', 'update', 'cancel'],
                    'workflows': ['create', 'read', 'update', 'execute', 'cancel'],
                    'browser_sessions': ['create', 'read', 'delete'],
                    'artifacts': ['read', 'delete'],
                    'analytics': ['read']
                })
            
            # Check admin permission
            if permissions.get('role') == 'admin':
                return True
            
            # Check specific resource permissions
            resource_permissions = permissions.get(resource, [])
            return action in resource_permissions
            
        except Exception as e:
            logger.error(f"Permission check failed: {e}")
            return False
    
    async def check_rate_limit(
        self, 
        organization: Organization, 
        endpoint: str
    ) -> RateLimitResult:
        """
        Check rate limits for organization and endpoint
        
        **Rate Limit Strategy:**
        - Per-organization limits
        - Per-endpoint specific limits
        - Sliding window algorithm
        - Different limits for different plan types
        
        **Limit Tiers:**
        - Free: 100 requests/hour
        - Pro: 1000 requests/hour  
        - Enterprise: 10000 requests/hour
        """
        
        if not self.redis:
            # No rate limiting if Redis not available
            return RateLimitResult(allowed=True)
        
        try:
            # Get rate limit configuration
            rate_limits = organization.settings.get('rate_limits', {})
            plan_type = organization.settings.get('plan_type', 'free')
            
            # Default limits by plan
            default_limits = {
                'free': {'requests_per_hour': 100, 'burst_limit': 20},
                'pro': {'requests_per_hour': 1000, 'burst_limit': 100},
                'enterprise': {'requests_per_hour': 10000, 'burst_limit': 500}
            }
            
            limits = rate_limits.get(endpoint, default_limits.get(plan_type, default_limits['free']))
            
            # Rate limit keys
            hour_key = f"rate_limit:{organization.organization_id}:{endpoint}:hour:{datetime.utcnow().hour}"
            burst_key = f"rate_limit:{organization.organization_id}:{endpoint}:burst"
            
            # Check hourly limit
            hourly_count = await self.redis.get(hour_key)
            hourly_count = int(hourly_count) if hourly_count else 0
            
            if hourly_count >= limits['requests_per_hour']:
                return RateLimitResult(
                    allowed=False,
                    limit=limits['requests_per_hour'],
                    remaining=0,
                    reset_time=datetime.utcnow().replace(minute=0, second=0, microsecond=0) + timedelta(hours=1),
                    retry_after=3600 - (datetime.utcnow().minute * 60 + datetime.utcnow().second)
                )
            
            # Check burst limit
            burst_count = await self.redis.get(burst_key)
            burst_count = int(burst_count) if burst_count else 0
            
            if burst_count >= limits['burst_limit']:
                return RateLimitResult(
                    allowed=False,
                    limit=limits['burst_limit'],
                    remaining=0,
                    reset_time=datetime.utcnow() + timedelta(minutes=1),
                    retry_after=60
                )
            
            # Increment counters
            pipe = self.redis.pipeline()
            pipe.incr(hour_key)
            pipe.expire(hour_key, 3600)
            pipe.incr(burst_key)
            pipe.expire(burst_key, 60)
            await pipe.execute()
            
            return RateLimitResult(
                allowed=True,
                limit=limits['requests_per_hour'],
                remaining=limits['requests_per_hour'] - hourly_count - 1,
                reset_time=datetime.utcnow().replace(minute=0, second=0, microsecond=0) + timedelta(hours=1)
            )
            
        except Exception as e:
            logger.error(f"Rate limit check failed: {e}")
            # Allow request if rate limiting fails
            return RateLimitResult(allowed=True)
    
    async def generate_api_key(
        self, 
        organization_id: str, 
        key_name: str,
        permissions: Dict[str, Any],
        expires_at: Optional[datetime] = None
    ) -> APIKeyResult:
        """
        Generate new API key for organization
        
        **Key Generation Process:**
        1. Generate cryptographically secure random key
        2. Hash key for storage
        3. Store in database with metadata
        4. Return unhashed key to user (only time it's visible)
        """
        
        try:
            # Generate API key
            api_key = self._generate_secure_api_key()
            api_key_hash = self._hash_api_key(api_key)
            
            # Create API key record
            api_key_record = APIKey(
                api_key_id=generate_api_key_id(),
                organization_id=organization_id,
                key_name=key_name,
                key_hash=api_key_hash,
                permissions=permissions,
                expires_at=expires_at,
                is_active=True,
                created_at=datetime.utcnow()
            )
            
            self.db.add(api_key_record)
            await self.db.commit()
            
            return APIKeyResult(
                success=True,
                api_key=api_key,  # Only returned once!
                api_key_id=api_key_record.api_key_id,
                expires_at=expires_at
            )
            
        except Exception as e:
            logger.error(f"API key generation failed: {e}")
            return APIKeyResult(
                success=False,
                error_message=str(e)
            )
    
    def _validate_api_key_format(self, api_key: str) -> bool:
        """Validate API key format"""
        
        # API key format: sk_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx (32 chars after prefix)
        if not api_key.startswith(('sk_live_', 'sk_test_')):
            return False
        
        if len(api_key) != 40:  # 8 char prefix + 32 char key
            return False
        
        # Check if key part is alphanumeric
        key_part = api_key[8:]
        return key_part.isalnum()
    
    def _hash_api_key(self, api_key: str) -> str:
        """Hash API key for secure storage"""
        
        # Use PBKDF2 with SHA-256
        kdf = PBKDF2HMAC(
            algorithm=hashes.SHA256(),
            length=32,
            salt=b'skyvern_api_salt',  # In production, use random salt per key
            iterations=100000,
        )
        
        key_hash = kdf.derive(api_key.encode('utf-8'))
        return base64.b64encode(key_hash).decode('utf-8')
    
    def _generate_secure_api_key(self) -> str:
        """Generate cryptographically secure API key"""
        
        # Generate 32 random bytes
        random_bytes = secrets.token_bytes(24)
        
        # Encode as base32 (no padding, URL-safe)
        key_part = base64.b32encode(random_bytes).decode('utf-8').rstrip('=').lower()
        
        return f"sk_live_{key_part}"
    
    async def _get_cached_organization(self, api_key: str) -> Optional[Organization]:
        """Get organization from cache"""
        
        if not self.redis:
            return None
        
        try:
            cache_key = f"auth:org:{self._hash_api_key(api_key)}"
            cached_data = await self.redis.get(cache_key)
            
            if cached_data:
                org_data = json.loads(cached_data)
                return Organization(**org_data)
            
            return None
            
        except Exception as e:
            logger.warning(f"Cache lookup failed: {e}")
            return None
    
    async def _cache_organization(self, api_key: str, organization: Organization):
        """Cache organization data"""
        
        if not self.redis:
            return
        
        try:
            cache_key = f"auth:org:{self._hash_api_key(api_key)}"
            org_data = organization.dict()
            
            await self.redis.setex(
                cache_key,
                self.api_key_cache_ttl,
                json.dumps(org_data, default=str)
            )
            
        except Exception as e:
            logger.warning(f"Cache write failed: {e}")
    
    async def _update_api_key_usage(self, api_key_id: str):
        """Update API key usage statistics"""
        
        try:
            await self.db.execute(
                text("""
                UPDATE api_keys 
                SET 
                    last_used_at = now(),
                    usage_count = usage_count + 1
                WHERE api_key_id = :api_key_id
                """),
                {"api_key_id": api_key_id}
            )
            
            await self.db.commit()
            
        except Exception as e:
            logger.warning(f"API key usage update failed: {e}")

class RateLimitResult(BaseModel):
    """Rate limit check result"""
    
    allowed: bool
    limit: Optional[int] = None
    remaining: Optional[int] = None
    reset_time: Optional[datetime] = None
    retry_after: Optional[int] = None

class APIKeyResult(BaseModel):
    """API key generation result"""
    
    success: bool
    api_key: Optional[str] = None
    api_key_id: Optional[str] = None
    expires_at: Optional[datetime] = None
    error_message: Optional[str] = None
```

---

## 🚀 Request/Response Flow Analysis

### Complete API Request Lifecycle

```mermaid
sequenceDiagram
    participant Client
    participant LoadBalancer
    participant FastAPI
    participant Middleware
    participant Auth
    participant RateLimit
    participant Handler
    participant Service
    participant Database
    participant Background
    
    Client->>LoadBalancer: HTTP Request
    LoadBalancer->>FastAPI: Route Request
    FastAPI->>Middleware: Process Middleware Stack
    
    Middleware->>Middleware: CORS Headers
    Middleware->>Middleware: Request ID
    Middleware->>Middleware: Timing Start
    Middleware->>Auth: Extract API Key
    
    Auth->>Database: Validate API Key
    Database-->>Auth: Organization Data
    Auth->>RateLimit: Check Limits
    RateLimit-->>Auth: Limit Status
    Auth-->>Middleware: Auth Result
    
    Middleware->>Handler: Route to Endpoint
    Handler->>Service: Business Logic
    Service->>Database: Data Operations
    Database-->>Service: Query Results
    Service-->>Handler: Service Response
    
    Handler->>Background: Queue Background Tasks
    Handler-->>Middleware: HTTP Response
    
    Middleware->>Middleware: Response Logging
    Middleware->>Middleware: Timing Headers
    Middleware-->>FastAPI: Final Response
    FastAPI-->>LoadBalancer: HTTP Response
    LoadBalancer-->>Client: Final Response
    
    Background->>Background: Execute Async Tasks
```

### Response Format Standardization

#### Success Response Format
```json
{
    "success": true,
    "data": {
        // Response payload specific to endpoint
    },
    "metadata": {
        "request_id": "req_123456789",
        "timestamp": "2024-01-15T10:30:00Z",
        "processing_time_ms": 245,
        "api_version": "v1"
    },
    "pagination": {  // Only for list endpoints
        "page": 1,
        "page_size": 20,
        "total_count": 150,
        "total_pages": 8,
        "has_next": true,
        "has_prev": false
    }
}
```

#### Error Response Format
```json
{
    "success": false,
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Request validation failed",
        "details": {
            "field_errors": [
                {
                    "field": "url",
                    "message": "URL must start with http:// or https://",
                    "code": "INVALID_URL_FORMAT"
                }
            ]
        }
    },
    "metadata": {
        "request_id": "req_123456789",
        "timestamp": "2024-01-15T10:30:00Z",
        "processing_time_ms": 12
    }
}
```

### Performance Optimization Strategies

#### 1. Response Caching
```python
from functools import wraps
import hashlib

def cache_response(ttl_seconds: int = 300):
    """Decorator for caching API responses"""
    
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Generate cache key from request parameters
            cache_key = _generate_cache_key(func.__name__, args, kwargs)
            
            # Try to get from cache
            if redis_client:
                cached = await redis_client.get(cache_key)
                if cached:
                    return json.loads(cached)
            
            # Execute function
            result = await func(*args, **kwargs)
            
            # Cache result
            if redis_client and result.get('success'):
                await redis_client.setex(
                    cache_key,
                    ttl_seconds,
                    json.dumps(result, default=str)
                )
            
            return result
        
        return wrapper
    return decorator

# Usage example
@router.get("/organizations/{org_id}/analytics")
@cache_response(ttl_seconds=600)  # Cache for 10 minutes
async def get_organization_analytics(...):
    # Expensive analytics calculation
    pass
```

#### 2. Database Query Optimization
```python
class OptimizedQueryService:
    """Service with optimized database queries"""
    
    async def get_tasks_with_steps_optimized(
        self, 
        organization_id: str,
        limit: int = 20
    ) -> List[Dict[str, Any]]:
        """Optimized query to get tasks with steps in single query"""
        
        # Use join instead of N+1 queries
        result = await self.db.execute(
            text("""
            SELECT 
                t.task_id,
                t.status as task_status,
                t.url,
                t.navigation_goal,
                t.created_at as task_created_at,
                t.completed_at as task_completed_at,
                
                -- Step information
                s.step_id,
                s.status as step_status,
                s.order as step_order,
                s.step_type,
                s.created_at as step_created_at,
                s.completed_at as step_completed_at,
                
                -- Action count per step
                (SELECT COUNT(*) FROM actions a WHERE a.step_id = s.step_id) as action_count
                
            FROM tasks t
            LEFT JOIN steps s ON t.task_id = s.task_id
            WHERE t.organization_id = :org_id
            ORDER BY t.created_at DESC, s.order ASC
            LIMIT :limit
            """),
            {"org_id": organization_id, "limit": limit}
        )
        
        # Group results by task
        tasks_dict = {}
        for row in result:
            task_id = row.task_id
            
            if task_id not in tasks_dict:
                tasks_dict[task_id] = {
                    "task_id": task_id,
                    "status": row.task_status,
                    "url": row.url,
                    "navigation_goal": row.navigation_goal,
                    "created_at": row.task_created_at,
                    "completed_at": row.task_completed_at,
                    "steps": []
                }
            
            if row.step_id:
                tasks_dict[task_id]["steps"].append({
                    "step_id": row.step_id,
                    "status": row.step_status,
                    "order": row.step_order,
                    "step_type": row.step_type,
                    "created_at": row.step_created_at,
                    "completed_at": row.step_completed_at,
                    "action_count": row.action_count
                })
        
        return list(tasks_dict.values())

#### 3. Async Background Processing
```python
from celery import Celery
from fastapi import BackgroundTasks

class BackgroundTaskManager:
    """Manager for background task processing"""
    
    def __init__(self):
        self.celery_app = Celery(
            'skyvern_tasks',
            broker='redis://localhost:6379/0',
            backend='redis://localhost:6379/0'
        )
    
    async def queue_task_execution(
        self, 
        task_id: str, 
        organization_id: str,
        priority: str = "normal"
    ):
        """Queue task for execution by automation engine"""
        
        try:
            # Add to execution queue with priority
            task_data = {
                "task_id": task_id,
                "organization_id": organization_id,
                "queued_at": datetime.utcnow().isoformat(),
                "priority": priority
            }
            
            # Use Celery for robust task queuing
            self.celery_app.send_task(
                'skyvern.automation.execute_task',
                args=[task_data],
                queue=f"priority_{priority}",
                priority=self._get_priority_value(priority)
            )
            
            logger.info(
                "Task queued for execution",
                extra={
                    "task_id": task_id,
                    "organization_id": organization_id,
                    "priority": priority
                }
            )
            
        except Exception as e:
            logger.error(f"Failed to queue task: {e}")
            raise
    
    def _get_priority_value(self, priority: str) -> int:
        """Convert priority string to numeric value"""
        priority_map = {
            "low": 1,
            "normal": 5,
            "high": 8,
            "urgent": 10
        }
        return priority_map.get(priority, 5)

# Background task functions
@celery_app.task
def send_webhook_notification(webhook_url: str, event_type: str, data: Dict[str, Any]):
    """Send webhook notification asynchronously"""
    
    try:
        import httpx
        
        payload = {
            "event": event_type,
            "data": data,
            "timestamp": datetime.utcnow().isoformat(),
            "source": "skyvern_api"
        }
        
        with httpx.Client(timeout=30) as client:
            response = client.post(
                webhook_url,
                json=payload,
                headers={"Content-Type": "application/json"}
            )
            
            if response.status_code >= 400:
                logger.warning(f"Webhook failed: {response.status_code}")
            else:
                logger.info(f"Webhook sent successfully: {event_type}")
                
    except Exception as e:
        logger.error(f"Webhook notification failed: {e}")
```

---

## 📊 API Documentation and Testing

### OpenAPI Specification Enhancement

```python
from fastapi.openapi.utils import get_openapi
from fastapi.openapi.docs import get_swagger_ui_html

def custom_openapi():
    """Generate enhanced OpenAPI specification"""
    
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="Skyvern API",
        version="2.0.0",
        description="""
        # Skyvern API Documentation
        
        Welcome to the Skyvern API! This API allows you to automate web interactions using AI-powered automation.
        
        ## Authentication
        
        All API endpoints require authentication using API keys. Include your API key in the `x-api-key` header:
        
        ```
        x-api-key: sk_live_your_api_key_here
        ```
        
        ## Rate Limits
        
        API requests are rate limited based on your organization's plan:
        - **Free**: 100 requests/hour
        - **Pro**: 1,000 requests/hour  
        - **Enterprise**: 10,000 requests/hour
        
        ## Webhooks
        
        Skyvern can send webhook notifications when tasks and workflows complete. Configure webhook URLs in your requests.
        
        ## SDKs and Libraries
        
        - **Python**: `pip install skyvern-python`
        - **Node.js**: `npm install skyvern-node`
        - **Go**: Coming soon
        
        ## Support
        
        - Documentation: https://docs.skyvern.com
        - Support: support@skyvern.com
        - Status Page: https://status.skyvern.com
        """,
        routes=app.routes,
    )
    
    # Add custom extensions
    openapi_schema["info"]["x-logo"] = {
        "url": "https://skyvern.com/logo.png"
    }
    
    # Add server information
    openapi_schema["servers"] = [
        {
            "url": "https://api.skyvern.com",
            "description": "Production server"
        },
        {
            "url": "https://staging-api.skyvern.com", 
            "description": "Staging server"
        }
    ]
    
    # Add security schemes
    openapi_schema["components"]["securitySchemes"] = {
        "ApiKeyAuth": {
            "type": "apiKey",
            "in": "header",
            "name": "x-api-key"
        }
    }
    
    # Apply security globally
    openapi_schema["security"] = [{"ApiKeyAuth": []}]
    
    # Add custom response examples
    _add_response_examples(openapi_schema)
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema

def _add_response_examples(schema: Dict[str, Any]):
    """Add response examples to OpenAPI schema"""
    
    # Task creation example
    if "paths" in schema and "/api/v1/tasks" in schema["paths"]:
        task_post = schema["paths"]["/api/v1/tasks"]["post"]
        task_post["responses"]["201"]["content"]["application/json"]["example"] = {
            "success": True,
            "data": {
                "task_id": "task_123456789",
                "organization_id": "org_abcdef123",
                "status": "created",
                "url": "https://example.com",
                "navigation_goal": "Fill out the contact form with test data",
                "created_at": "2024-01-15T10:30:00Z"
            },
            "metadata": {
                "request_id": "req_987654321",
                "timestamp": "2024-01-15T10:30:00Z"
            }
        }

app.openapi = custom_openapi
```

### API Testing Framework

```python
import pytest
from httpx import AsyncClient
from fastapi.testclient import TestClient

class APITestSuite:
    """Comprehensive API testing suite"""
    
    def __init__(self, app: FastAPI):
        self.app = app
        self.client = TestClient(app)
        self.api_key = "sk_test_" + "x" * 32  # Test API key
    
    @pytest.mark.asyncio
    async def test_task_creation_flow(self):
        """Test complete task creation and execution flow"""
        
        # Test data
        task_data = {
            "url": "https://httpbin.org/forms/post",
            "navigation_goal": "Fill out the form with test data",
            "data_extraction_goal": "Extract the confirmation message"
        }
        
        # Create task
        response = self.client.post(
            "/api/v1/tasks",
            json=task_data,
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 201
        task_response = response.json()
        assert task_response["success"] is True
        assert "task_id" in task_response["data"]
        
        task_id = task_response["data"]["task_id"]
        
        # Get task details
        response = self.client.get(
            f"/api/v1/tasks/{task_id}",
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 200
        task_details = response.json()
        assert task_details["data"]["task_id"] == task_id
        assert task_details["data"]["status"] in ["created", "queued", "running"]
        
        # Test task cancellation
        response = self.client.post(
            f"/api/v1/tasks/{task_id}/cancel",
            json={"reason": "Test cancellation"},
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 200
        cancel_response = response.json()
        assert cancel_response["data"]["status"] == "cancelled"
    
    @pytest.mark.asyncio
    async def test_workflow_execution_flow(self):
        """Test workflow creation and execution"""
        
        # Workflow definition
        workflow_data = {
            "title": "Test Workflow",
            "description": "A test workflow for API testing",
            "workflow_definition": {
                "blocks": [
                    {
                        "id": "nav_1",
                        "type": "navigation",
                        "parameters": {
                            "url": "https://httpbin.org/forms/post",
                            "navigation_goal": "Navigate to the form"
                        }
                    },
                    {
                        "id": "extract_1", 
                        "type": "extraction",
                        "parameters": {
                            "data_extraction_goal": "Extract form fields"
                        }
                    }
                ],
                "connections": [
                    {
                        "source": "nav_1",
                        "target": "extract_1"
                    }
                ]
            },
            "parameter_schema": {
                "type": "object",
                "properties": {
                    "test_param": {
                        "type": "string",
                        "description": "Test parameter"
                    }
                }
            }
        }
        
        # Create workflow
        response = self.client.post(
            "/api/v1/workflows",
            json=workflow_data,
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 201
        workflow_response = response.json()
        workflow_id = workflow_response["data"]["workflow_id"]
        
        # Execute workflow
        execution_data = {
            "parameters": {
                "test_param": "test_value"
            },
            "priority": "normal"
        }
        
        response = self.client.post(
            f"/api/v1/workflows/{workflow_id}/runs",
            json=execution_data,
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 201
        run_response = response.json()
        run_id = run_response["data"]["workflow_run_id"]
        
        # Get workflow run status
        response = self.client.get(
            f"/api/v1/workflows/{workflow_id}/runs/{run_id}",
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 200
        run_details = response.json()
        assert run_details["data"]["workflow_run_id"] == run_id
    
    @pytest.mark.asyncio
    async def test_authentication_and_authorization(self):
        """Test API authentication and authorization"""
        
        # Test without API key
        response = self.client.get("/api/v1/tasks")
        assert response.status_code == 401
        
        # Test with invalid API key
        response = self.client.get(
            "/api/v1/tasks",
            headers={"x-api-key": "invalid_key"}
        )
        assert response.status_code == 401
        
        # Test with valid API key
        response = self.client.get(
            "/api/v1/tasks",
            headers={"x-api-key": self.api_key}
        )
        assert response.status_code == 200
    
    @pytest.mark.asyncio
    async def test_rate_limiting(self):
        """Test API rate limiting"""
        
        # Make rapid requests to trigger rate limiting
        responses = []
        for i in range(150):  # Exceed free tier limit
            response = self.client.get(
                "/api/v1/tasks",
                headers={"x-api-key": self.api_key}
            )
            responses.append(response.status_code)
        
        # Should eventually get rate limited
        assert 429 in responses
    
    @pytest.mark.asyncio
    async def test_error_handling(self):
        """Test API error handling"""
        
        # Test validation errors
        invalid_task_data = {
            "url": "not_a_url",  # Invalid URL
            "navigation_goal": ""  # Empty goal
        }
        
        response = self.client.post(
            "/api/v1/tasks",
            json=invalid_task_data,
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 422
        error_response = response.json()
        assert error_response["success"] is False
        assert "error" in error_response
        assert "VALIDATION_ERROR" in error_response["error"]["code"]
    
    @pytest.mark.asyncio
    async def test_pagination(self):
        """Test API pagination"""
        
        response = self.client.get(
            "/api/v1/tasks?page=1&page_size=10",
            headers={"x-api-key": self.api_key}
        )
        
        assert response.status_code == 200
        paginated_response = response.json()
        assert "pagination" in paginated_response["data"]
        
        pagination = paginated_response["data"]["pagination"]
        assert "page" in pagination
        assert "page_size" in pagination
        assert "total_count" in pagination
        assert "total_pages" in pagination

# Performance testing
class APIPerformanceTests:
    """API performance and load testing"""
    
    @pytest.mark.asyncio
    async def test_concurrent_requests(self):
        """Test API performance under concurrent load"""
        
        import asyncio
        import time
        
        async def make_request():
            async with AsyncClient(app=app, base_url="http://test") as client:
                response = await client.get(
                    "/api/v1/tasks",
                    headers={"x-api-key": self.api_key}
                )
                return response.status_code
        
        # Test 50 concurrent requests
        start_time = time.time()
        tasks = [make_request() for _ in range(50)]
        results = await asyncio.gather(*tasks)
        end_time = time.time()
        
        # All requests should succeed
        assert all(status == 200 for status in results)
        
        # Should complete within reasonable time
        duration = end_time - start_time
        assert duration < 10  # Should complete within 10 seconds
        
        # Calculate requests per second
        rps = len(tasks) / duration
        assert rps > 5  # Should handle at least 5 RPS
```

---

## ✅ Phase 3 Completion Checklist

### Understanding Verification

- [ ] **FastAPI Architecture**: Can explain the complete application structure and middleware stack
- [ ] **Endpoint Design**: Understand all REST endpoints, their parameters, and response formats
- [ ] **Authentication Flow**: Know how API key authentication works end-to-end
- [ ] **Request Processing**: Can trace a request through the entire middleware pipeline
- [ ] **Error Handling**: Understand comprehensive error handling and HTTP status mapping
- [ ] **Performance Optimization**: Know caching, rate limiting, and query optimization strategies

### Practical Skills

- [ ] **API Development**: Can create new endpoints following established patterns
- [ ] **Authentication Setup**: Can implement API key generation and validation
- [ ] **Testing**: Can write comprehensive API tests covering all scenarios
- [ ] **Documentation**: Can enhance OpenAPI specifications and examples
- [ ] **Debugging**: Can troubleshoot API issues using logs and monitoring
- [ ] **Performance Tuning**: Can optimize API performance and handle scaling

### Key Questions to Answer

1. **How does the authentication system work?**
   - API keys are hashed and stored securely in the database
   - Each request validates the API key and loads organization context
   - Permissions are checked based on organization settings
   - Rate limiting is applied per organization and endpoint

2. **What is the complete request lifecycle?**
   - Request arrives at load balancer/reverse proxy
   - FastAPI processes through middleware stack (CORS, auth, rate limiting, logging)
   - Request is routed to appropriate endpoint handler
   - Business logic is executed via service layer
   - Database operations are performed
   - Response is formatted and returned through middleware
   - Background tasks are queued for async processing

3. **How are errors handled consistently?**
   - Global exception handlers catch all error types
   - Skyvern-specific exceptions map to appropriate HTTP status codes
   - Validation errors provide detailed field-level feedback
   - All errors include request IDs for tracing
   - Error responses follow standardized format

4. **What performance optimizations are implemented?**
   - Response caching for expensive operations
   - Database query optimization with joins and indexes
   - Rate limiting to prevent abuse
   - Background task processing for heavy operations
   - Connection pooling for database efficiency

---

## 🚀 Ready for Phase 4?

You now have comprehensive knowledge of Skyvern's API & Web Framework, including:

- **Complete FastAPI architecture** with middleware stack and dependency injection
- **Authentication and authorization** systems with API key management
- **All REST endpoints** for tasks, workflows, and system management
- **Performance optimization** strategies and monitoring
- **Testing frameworks** for API quality assurance

**Next Phase Preview**: Phase 4 will explore the **Core Automation Engine**, covering:
- Main automation orchestrator and agent architecture
- Task execution flow and step management
- AI integration and decision making
- Browser state coordination and management
- Error handling and recovery mechanisms

The API knowledge you've gained shows how external clients interact with Skyvern, and Phase 4 will dive into the automation engine that powers these APIs.

Type "ready for phase 4" when you want to continue with the Core Automation Engine deep dive!# Phase 3: API & Web Framework
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