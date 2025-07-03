# 🔗 Phase 11.1: API Integration & Client Architecture

## 📋 **API Layer Overview**

The Skyvern frontend implements a sophisticated API client system for seamless backend communication.

---

## 🏗️ **API Client Architecture**

### **Core API Structure**

```mermaid
graph TB
    subgraph "API Layer"
        A[AxiosClient.ts] --> B[Credential Getter]
        A --> C[Request Interceptors]
        A --> D[Response Interceptors]
        
        E[types.ts] --> F[API Response Types]
        E --> G[Request Types]
        E --> H[Domain Models]
        
        I[QueryClient.ts] --> J[TanStack Query Setup]
        I --> K[Cache Configuration]
        I --> L[Error Handling]
    end
    
    subgraph "Authentication"
        M[Environment Variables] --> N[API Keys]
        M --> O[Base URLs]
        P[useCredentialGetter Hook] --> Q[Token Management]
    end
    
    B --> P
    F --> R[Components]
```

---

## 🔐 **Authentication System**

### **Credential Management Flow**

```mermaid
sequenceDiagram
    participant C as Component
    participant H as useCredentialGetter Hook
    participant A as AxiosClient
    participant B as Backend API
    
    C->>H: Request credentials
    H->>H: Check environment type
    alt Cloud Environment
        H->>H: Return cloud credentials
    else Local Environment  
        H->>H: Return local API key
    end
    H->>C: Return credential getter
    C->>A: Make API call with credentials
    A->>B: HTTP Request with auth headers
    B->>A: Response
    A->>C: Typed response data
```

### **Environment Configuration**

```typescript
// Environment-based credential system
interface CredentialGetter {
  apiKey?: string;
  organization?: string;
  environment: 'local' | 'cloud';
}

// Usage in components
const credentialGetter = useCredentialGetter();
const client = await getClient(credentialGetter);
```

---

## 📝 **Type System Architecture**

### **Core API Types Structure**

```mermaid
classDiagram
    class TaskApiResponse {
        +task_id: string
        +status: Status
        +request: TaskRequest
        +created_at: string
        +modified_at: string
    }
    
    class WorkflowApiResponse {
        +workflow_id: string
        +title: string
        +workflow_definition: WorkflowDefinition
        +is_saved_task: boolean
    }
    
    class StepApiResponse {
        +step_id: string
        +task_id: string
        +status: Status
        +output: StepOutput
        +order: number
    }
    
    class ArtifactApiResponse {
        +artifact_id: string
        +artifact_type: ArtifactType
        +uri: string
        +step_id: string
    }
    
    TaskApiResponse --> StepApiResponse
    StepApiResponse --> ArtifactApiResponse
```

### **Status Enumeration System**

```typescript
export enum Status {
  Created = "created",
  Running = "running", 
  Failed = "failed",
  Terminated = "terminated",
  Completed = "completed",
  TimedOut = "timed_out",
  Canceled = "canceled"
}

// Type guards for status checking
export function statusIsNotFinalized(status: Status): boolean {
  return ![Status.Completed, Status.Failed, Status.Terminated, 
           Status.TimedOut, Status.Canceled].includes(status);
}
```

---

## 🔄 **Query Client Configuration**

### **TanStack Query Setup**

```mermaid
graph LR
    A[QueryClient] --> B[Default Options]
    A --> C[Cache Configuration]  
    A --> D[Error Handling]
    
    B --> E[5 minute stale time]
    B --> F[10 minute cache time]
    B --> G[3 retry attempts]
    
    C --> H[Optimistic Updates]
    C --> I[Background Refetch]
    C --> J[Window Focus Refetch]
    
    D --> K[Toast Notifications]
    D --> L[Error Boundaries]
    D --> M[Fallback UI]
```

### **Query Configuration Details**

```typescript
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,      // 5 minutes
      gcTime: 10 * 60 * 1000,        // 10 minutes  
      retry: 3,
      refetchOnWindowFocus: true,
      refetchOnReconnect: true
    },
    mutations: {
      retry: 1,
      onError: (error) => {
        toast({
          title: "Operation Failed",
          description: error.message,
          variant: "destructive"
        });
      }
    }
  }
});
```

---

## 🌐 **API Endpoints Structure**

### **Endpoint Categories**

```mermaid
mindmap
  root((API Endpoints))
    Tasks
      GET /tasks
      POST /tasks
      GET /tasks/:id
      GET /tasks/:id/steps
      GET /tasks/:id/artifacts
    Workflows
      GET /workflows
      POST /workflows  
      GET /workflows/:id
      POST /workflows/:id/run
      GET /workflows/runs/:id
    Authentication
      POST /auth/login
      GET /auth/profile
      POST /auth/refresh
    Organizations
      GET /organizations
      POST /organizations
      GET /organizations/:id/users
    Artifacts
      GET /artifacts/:id
      POST /artifacts/upload
      GET /artifacts/download/:id
```

### **Request/Response Patterns**

```typescript
// Standardized API patterns
interface BaseApiResponse {
  created_at: string;
  modified_at: string;
}

interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  size: number;
}

interface ErrorResponse {
  error: string;
  message: string;
  details?: Record<string, any>;
}
```

---

## 🎯 **Custom Hooks Pattern**

### **API Hook Architecture**

```mermaid
graph TB
    A[useCredentialGetter] --> B[API Hooks]
    B --> C[useTaskQuery]
    B --> D[useWorkflowQuery] 
    B --> E[useStepsQuery]
    B --> F[useArtifactsQuery]
    
    G[Mutation Hooks] --> H[useCreateTask]
    G --> I[useRunWorkflow]
    G --> J[useUpdateWorkflow]
    
    C --> K[Components]
    D --> K
    E --> K
    F --> K
    H --> K
    I --> K
    J --> K
```

### **Hook Implementation Example**

```typescript
// Custom query hook with error handling
export function useTaskQuery(taskId: string) {
  const credentialGetter = useCredentialGetter();
  
  return useQuery<TaskApiResponse>({
    queryKey: ["task", taskId],
    queryFn: async () => {
      const client = await getClient(credentialGetter);
      return client.get(`/tasks/${taskId}`)
        .then(response => response.data);
    },
    enabled: !!taskId,
    staleTime: 30 * 1000, // 30 seconds for real-time data
    refetchInterval: (data) => 
      statusIsNotFinalized(data?.status) ? 5000 : false
  });
}
```

---

## 🛡️ **Error Handling Strategy**

### **Error Handling Flow**

```mermaid
sequenceDiagram
    participant C as Component
    participant Q as Query Hook
    participant A as API Client
    participant E as Error Handler
    participant U as UI Toast
    
    C->>Q: Trigger API call
    Q->>A: HTTP Request
    A->>A: Network/Response error
    A->>E: Error interceptor
    E->>E: Parse error type
    alt Validation Error
        E->>U: Show field errors
    else Network Error
        E->>U: Show connectivity message  
    else Auth Error
        E->>E: Redirect to login
    else Server Error
        E->>U: Show generic error
    end
    E->>Q: Return error state
    Q->>C: Update loading/error state
```

### **Error Types & Handling**

```typescript
// Comprehensive error handling
interface ApiError {
  status: number;
  message: string;
  details?: Record<string, string[]>;
}

// Error categorization
function handleApiError(error: AxiosError): void {
  if (error.response?.status === 401) {
    // Authentication error - redirect to login
    window.location.href = '/login';
  } else if (error.response?.status === 422) {
    // Validation error - show field errors
    showValidationErrors(error.response.data.details);
  } else if (error.response?.status >= 500) {
    // Server error - show generic message
    toast({
      title: "Server Error",
      description: "Please try again later",
      variant: "destructive"
    });
  }
}
```

---

## 📊 **Real-time Data Handling**

### **Polling Strategy for Live Updates**

```mermaid
graph LR
    A[Running Task] --> B{Status Check}
    B -->|Still Running| C[Poll Every 5s]
    B -->|Completed| D[Stop Polling]
    B -->|Failed| E[Stop Polling]
    
    C --> F[Update UI]
    F --> A
    
    D --> G[Final UI State]
    E --> G
```

### **WebSocket Integration**

```typescript
// Real-time updates for automation streams
interface StreamMessage {
  task_id: string;
  status: string;
  screenshot?: string;
  action_taken?: string;
}

// WebSocket connection for live browser streaming
const wssBaseUrl = import.meta.env.VITE_WSS_BASE_URL;
const ws = new WebSocket(`${wssBaseUrl}/stream/${taskId}`);
```

---

## 🎯 **Key Implementation Insights**

### **Best Practices Applied**

1. **Type Safety** - Full TypeScript coverage for API contracts
2. **Error Boundaries** - Graceful error handling at all levels  
3. **Caching Strategy** - Optimized data fetching and updates
4. **Real-time Updates** - Polling and WebSocket integration
5. **Environment Flexibility** - Supports local and cloud deployments
6. **Performance** - Efficient query invalidation and background updates

### **Architecture Benefits**

- **Maintainable** - Clear separation of concerns
- **Testable** - Isolated API logic with dependency injection
- **Scalable** - Easy to add new endpoints and features
- **Reliable** - Comprehensive error handling and retry logic

---

*This API integration layer provides the foundation for all frontend-backend communication in the Skyvern application.*