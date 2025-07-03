# 🧩 Phase 11.2: Component Architecture & UI System

## 🎨 **Component System Overview**

Skyvern's frontend employs a sophisticated component architecture built on Shadcn/UI and Radix primitives.

---

## 🏗️ **Component Hierarchy**

### **Three-Layer Architecture**

```mermaid
graph TB
    subgraph "Application Layer"
        A[Page Components] --> B[Route-specific Logic]
        A --> C[Business Logic Integration]
    end
    
    subgraph "Shared Components Layer"  
        D[Custom Components] --> E[Domain-specific Logic]
        D --> F[Composite Behaviors]
    end
    
    subgraph "UI Foundation Layer"
        G[Shadcn/UI Components] --> H[Design System]
        I[Radix Primitives] --> J[Accessibility]
        K[Tailwind Classes] --> L[Styling System]
    end
    
    A --> D
    D --> G
    D --> I
    G --> K
```

---

## 🔧 **Core UI Components**

### **Form & Input Components**

```mermaid
classDiagram
    class Input {
        +value: string
        +onChange: function
        +placeholder: string
        +disabled: boolean
        +error: boolean
    }
    
    class Textarea {
        +value: string
        +onChange: function
        +rows: number
        +autoResize: boolean
    }
    
    class Select {
        +value: string
        +onValueChange: function
        +options: Option[]
        +placeholder: string
    }
    
    class Checkbox {
        +checked: boolean
        +onCheckedChange: function
        +disabled: boolean
    }
    
    class Switch {
        +checked: boolean
        +onCheckedChange: function
        +disabled: boolean
    }
    
    Input --> FormField
    Textarea --> FormField
    Select --> FormField
    Checkbox --> FormField
    Switch --> FormField
```

### **Display & Feedback Components**

```mermaid
classDiagram
    class Badge {
        +variant: default | secondary | destructive
        +size: sm | default | lg
        +children: ReactNode
    }
    
    class StatusBadge {
        +status: Status
        +showIcon: boolean
        +getVariant(): BadgeVariant
    }
    
    class Skeleton {
        +className: string
        +animate: boolean
    }
    
    class Toast {
        +title: string
        +description: string
        +variant: default | destructive
        +duration: number
    }
    
    class Dialog {
        +open: boolean
        +onOpenChange: function
        +children: ReactNode
    }
    
    StatusBadge --> Badge
    Toast --> ToastProvider
    Dialog --> DialogContent
```

---

## 🎯 **Custom Component Patterns**

### **Auto-Resizing Textarea**

```mermaid
sequenceDiagram
    participant U as User
    participant C as AutoResizingTextarea
    participant R as Ref
    participant D as DOM
    
    U->>C: Type content
    C->>R: Get textarea ref
    R->>D: Calculate scrollHeight
    D->>C: Return height value
    C->>C: Adjust min-height style
    C->>U: Render resized textarea
```

```typescript
// Implementation pattern
export function AutoResizingTextarea({ value, onChange, ...props }) {
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  
  useLayoutEffect(() => {
    const textarea = textareaRef.current;
    if (textarea) {
      textarea.style.height = 'auto';
      textarea.style.height = `${textarea.scrollHeight}px`;
    }
  }, [value]);
  
  return (
    <Textarea
      ref={textareaRef}
      value={value}
      onChange={onChange}
      className="resize-none overflow-hidden"
      {...props}
    />
  );
}
```

### **Zoomable Image Component**

```mermaid
stateDiagram-v2
    [*] --> Normal : Initial Load
    Normal --> Zoomed : Click to Zoom
    Zoomed --> Normal : Click to Close
    Zoomed --> Panning : Drag Image
    Panning --> Zoomed : Release Drag
    
    Normal : Display image at fit size
    Zoomed : Full resolution overlay
    Panning : Mouse/touch interaction
```

---

## 📊 **Data Display Components**

### **Table System Architecture**

```mermaid
graph TB
    subgraph "Table Components"
        A[Table] --> B[TableHeader]
        A --> C[TableBody]  
        A --> D[TableFooter]
        
        B --> E[TableRow]
        C --> E
        E --> F[TableHead]
        E --> G[TableCell]
    end
    
    subgraph "Table Features"
        H[Sorting] --> I[Column Headers]
        J[Pagination] --> K[Footer Controls]
        L[Selection] --> M[Checkbox Column]
        N[Responsive] --> O[Mobile Breakpoints]
    end
    
    A --> H
    A --> J
    A --> L
    A --> N
```

### **Card System Pattern**

```typescript
// Flexible card composition
interface CardProps {
  children: ReactNode;
  className?: string;
  variant?: 'default' | 'elevated' | 'outlined';
}

// Usage pattern in lists
function TaskCard({ task }: { task: TaskApiResponse }) {
  return (
    <Card className="hover:shadow-lg transition-shadow">
      <CardHeader>
        <CardTitle>{task.request.navigation_goal}</CardTitle>
        <CardDescription>
          <StatusBadge status={task.status} />
        </CardDescription>
      </CardHeader>
      <CardContent>
        <p>URL: {task.request.url}</p>
        <p>Created: {timeFormatWithShortDate(task.created_at)}</p>
      </CardContent>
    </Card>
  );
}
```

---

## 🎨 **Design System Implementation**

### **Theme Configuration**

```mermaid
graph LR
    A[CSS Variables] --> B[Light Theme]
    A --> C[Dark Theme]
    
    B --> D[Background Colors]
    B --> E[Text Colors]
    B --> F[Border Colors]
    
    C --> G[Background Colors]
    C --> H[Text Colors]  
    C --> I[Border Colors]
    
    D --> J[Component Styles]
    E --> J
    F --> J
    G --> J
    H --> J
    I --> J
```

### **Color System Structure**

```css
:root {
  /* Base colors */
  --background: 0 0% 100%;
  --foreground: 222.2 84% 4.9%;
  
  /* Component colors */
  --card: 0 0% 100%;
  --card-foreground: 222.2 84% 4.9%;
  
  --popover: 0 0% 100%;
  --popover-foreground: 222.2 84% 4.9%;
  
  /* State colors */
  --primary: 222.2 47.4% 11.2%;
  --primary-foreground: 210 40% 98%;
  
  --secondary: 210 40% 96%;
  --secondary-foreground: 222.2 84% 4.9%;
  
  --destructive: 0 84.2% 60.2%;
  --destructive-foreground: 210 40% 98%;
}

.dark {
  --background: 222.2 84% 4.9%;
  --foreground: 210 40% 98%;
  /* ... dark theme overrides */
}
```

---

## 🔄 **State Management in Components**

### **Local State Patterns**

```mermaid
graph TB
    A[useState] --> B[Simple State]
    C[useReducer] --> D[Complex State]
    E[useRef] --> F[DOM References]
    G[useCallback] --> H[Memoized Functions]
    I[useMemo] --> J[Computed Values]
    
    B --> K[Form Inputs]
    B --> L[UI Toggles]
    D --> M[Multi-step Forms]
    D --> N[Complex Workflows]
    F --> O[Focus Management]
    F --> P[Animation Refs]
```

### **Global State Integration**

```typescript
// Store integration pattern
function WorkflowEditor() {
  // Global stores
  const { collapsed, setCollapsed } = useSidebarStore();
  const { environment } = useSettingsStore();
  const { clientId } = useClientIdStore();
  
  // Server state
  const { data: workflow, isLoading } = useWorkflowQuery(workflowId);
  
  // Local state
  const [selectedNode, setSelectedNode] = useState<string | null>(null);
  
  return (
    <div className={cn("flex", collapsed && "sidebar-collapsed")}>
      {/* Component implementation */}
    </div>
  );
}
```

---

## 🎪 **Specialized Components**

### **Browser Stream Component**

```mermaid
sequenceDiagram
    participant C as BrowserStream
    participant W as WebSocket
    participant V as VNC Client
    participant U as UI
    
    C->>W: Connect to stream
    W->>C: Connection established
    C->>V: Initialize VNC client
    V->>U: Render browser view
    
    loop Real-time Updates
        W->>C: Screenshot data
        C->>V: Update display
        V->>U: Refresh view
    end
    
    U->>C: User interaction
    C->>W: Send control commands
    W->>C: Acknowledge command
```

### **Code Editor Integration**

```typescript
// Monaco Editor wrapper
interface CodeEditorProps {
  value: string;
  onChange?: (value: string) => void;
  language: 'json' | 'python' | 'javascript';
  readOnly?: boolean;
  height?: string;
}

function CodeEditor({ value, onChange, language, readOnly, height = "200px" }: CodeEditorProps) {
  return (
    <div className="border rounded-md overflow-hidden">
      <MonacoEditor
        height={height}
        language={language}
        value={value}
        onChange={onChange}
        options={{
          readOnly,
          minimap: { enabled: false },
          lineNumbers: 'on',
          wordWrap: 'on',
          theme: 'vs-dark'
        }}
      />
    </div>
  );
}
```

---

## 📱 **Responsive Design Strategy**

### **Breakpoint System**

```mermaid
graph LR
    A[Mobile] --> B[640px+]
    B[Tablet] --> C[768px+] 
    C[Desktop] --> D[1024px+]
    D[Large] --> E[1280px+]
    E[XL] --> F[1536px+]
    
    A --> G[Stack Layout]
    B --> H[Sidebar Collapse]
    C --> I[Full Layout]
    D --> J[Multi-column]
    E --> K[Wide Margins]
```

### **Responsive Patterns**

```typescript
// Responsive component pattern
function TaskDetail() {
  return (
    <div className="flex flex-col lg:flex-row gap-6">
      {/* Main content - full width on mobile, 2/3 on desktop */}
      <div className="w-full lg:w-2/3 space-y-4">
        <TaskOverview />
        <TaskSteps />
      </div>
      
      {/* Sidebar - bottom on mobile, 1/3 on desktop */}  
      <div className="w-full lg:w-1/3">
        <TaskSidebar />
      </div>
    </div>
  );
}
```

---

## ♿ **Accessibility Implementation**

### **ARIA Integration**

```mermaid
graph TB
    A[Radix Primitives] --> B[Built-in ARIA]
    B --> C[Screen Reader Support]
    B --> D[Keyboard Navigation]
    B --> E[Focus Management]
    
    F[Custom Components] --> G[ARIA Labels]
    F --> H[Role Attributes]
    F --> I[State Indicators]
    
    C --> J[Inclusive UX]
    D --> J
    E --> J
    G --> J
    H --> J
    I --> J
```

### **Keyboard Navigation**

```typescript
// Accessible modal pattern
function AccessibleDialog({ children, ...props }) {
  return (
    <Dialog {...props}>
      <DialogContent className="focus:outline-none">
        <DialogHeader>
          <DialogTitle>Dialog Title</DialogTitle>
          <DialogDescription>
            Dialog description for screen readers
          </DialogDescription>
        </DialogHeader>
        
        <div className="focus-within:ring-2 focus-within:ring-primary">
          {children}
        </div>
        
        <DialogFooter>
          <Button variant="outline" onClick={onCancel}>
            Cancel
          </Button>
          <Button onClick={onConfirm}>
            Confirm
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

---

## 🎯 **Performance Optimization**

### **Component Optimization Strategies**

```mermaid
graph TB
    A[React.memo] --> B[Prevent Re-renders]
    C[useMemo] --> D[Expensive Calculations]
    E[useCallback] --> F[Stable References]
    G[Lazy Loading] --> H[Code Splitting]
    I[Virtual Scrolling] --> J[Large Lists]
    
    B --> K[Performance Gains]
    D --> K
    F --> K
    H --> K
    J --> K
```

### **Lazy Loading Implementation**

```typescript
// Route-based code splitting
const TaskDetail = lazy(() => import('./routes/tasks/detail/TaskDetail'));
const WorkflowEditor = lazy(() => import('./routes/workflows/editor/WorkflowEditor'));

function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <Routes>
        <Route path="/tasks/:taskId" element={<TaskDetail />} />
        <Route path="/workflows/:workflowId/edit" element={<WorkflowEditor />} />
      </Routes>
    </Suspense>
  );
}

// Component-level lazy loading
const HeavyChart = lazy(() => import('./components/HeavyChart'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <Button onClick={() => setShowChart(true)}>
        Load Chart
      </Button>
      
      {showChart && (
        <Suspense fallback={<Skeleton className="h-64" />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```

---

## 🔧 **Component Testing Patterns**

### **Testing Strategy**

```mermaid
graph LR
    A[Unit Tests] --> B[Individual Components]
    C[Integration Tests] --> D[Component Groups]
    E[Visual Tests] --> F[UI Consistency]
    G[Accessibility Tests] --> H[A11y Compliance]
    
    B --> I[Jest + Testing Library]
    D --> I
    F --> J[Storybook]
    H --> K[axe-core]
```

### **Component Test Example**

```typescript
// Component testing pattern
import { render, screen, fireEvent } from '@testing-library/react';
import { StatusBadge } from './StatusBadge';
import { Status } from '@/api/types';

describe('StatusBadge', () => {
  it('renders correct variant for completed status', () => {
    render(<StatusBadge status={Status.Completed} />);
    
    const badge = screen.getByRole('status');
    expect(badge).toHaveClass('bg-green-100');
    expect(badge).toHaveTextContent('completed');
  });
  
  it('shows loading state for running status', () => {
    render(<StatusBadge status={Status.Running} />);
    
    const badge = screen.getByRole('status');
    expect(badge).toHaveClass('bg-blue-100');
    expect(badge).toHaveAttribute('aria-label', 'Task is running');
  });
});
```

---

## 🎨 **Advanced Component Patterns**

### **Compound Component Pattern**

```typescript
// Flexible API design
interface CardApi {
  Card: React.FC<CardProps>;
  Header: React.FC<CardHeaderProps>;
  Title: React.FC<CardTitleProps>;
  Description: React.FC<CardDescriptionProps>;
  Content: React.FC<CardContentProps>;
  Footer: React.FC<CardFooterProps>;
}

// Usage provides flexibility
function TaskCard() {
  return (
    <Card.Card>
      <Card.Header>
        <Card.Title>Task Name</Card.Title>
        <Card.Description>
          <StatusBadge status={task.status} />
        </Card.Description>
      </Card.Header>
      <Card.Content>
        <TaskDetails />
      </Card.Content>
      <Card.Footer>
        <TaskActions />
      </Card.Footer>
    </Card.Card>
  );
}
```

### **Render Props Pattern**

```typescript
// Flexible data fetching component
interface DataFetcherProps<T> {
  queryKey: string[];
  queryFn: () => Promise<T>;
  children: (props: {
    data: T | undefined;
    isLoading: boolean;
    error: Error | null;
  }) => ReactNode;
}

function DataFetcher<T>({ queryKey, queryFn, children }: DataFetcherProps<T>) {
  const { data, isLoading, error } = useQuery({ queryKey, queryFn });
  
  return <>{children({ data, isLoading, error })}</>;
}

// Usage
function TaskList() {
  return (
    <DataFetcher
      queryKey={['tasks']}
      queryFn={fetchTasks}
    >
      {({ data: tasks, isLoading, error }) => {
        if (isLoading) return <TaskListSkeleton />;
        if (error) return <ErrorMessage error={error} />;
        return <TaskGrid tasks={tasks} />;
      }}
    </DataFetcher>
  );
}
```

---

## 🎯 **Key Architecture Benefits**

### **Maintainability Features**

1. **Consistent Design System** - Unified look and feel
2. **Reusable Components** - DRY principle applied
3. **Type Safety** - Full TypeScript integration
4. **Accessibility Built-in** - WCAG compliance by default
5. **Performance Optimized** - Lazy loading and memoization
6. **Testing Ready** - Testable component structure

### **Developer Experience**

- **Hot Reloading** - Fast development cycle
- **Component Documentation** - Self-documenting props
- **Error Boundaries** - Graceful error handling
- **DevTools Integration** - React and Zustand devtools
- **Linting & Formatting** - Consistent code style

### **Scalability Considerations**

- **Modular Architecture** - Easy to extend and modify
- **Separation of Concerns** - Clear responsibility boundaries
- **State Management** - Predictable data flow
- **Code Splitting** - Efficient bundle management
- **Component Library** - Reusable across projects

---

*This component architecture provides a robust foundation for building scalable, maintainable, and accessible user interfaces in the Skyvern application.*