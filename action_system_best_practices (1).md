# 🎯 Best Practices & Patterns
*Mastering the Skyvern Action System*

---

## 🏆 Action Design Principles

### 1. **Single Responsibility**
Each action should have one clear purpose.

```python
# ✅ Good: Single responsibility
class ClickAction(WebAction):
    action_type: Literal[ActionType.CLICK] = ActionType.CLICK
    
# ❌ Bad: Multiple responsibilities
class ClickAndWaitAction(WebAction):
    # Combines clicking and waiting - should be separate actions
```

### 2. **Immutable Action State**
Actions should be immutable once created.

```python
# ✅ Good: Create new action for modifications
original_action = ClickAction(element_id="btn-1")
modified_action = original_action.model_copy(
    update={"confidence_float": 0.95}
)

# ❌ Bad: Modifying existing action
original_action.confidence_float = 0.95  # Don't do this
```

### 3. **Explicit Error Handling**
Always handle potential failures gracefully.

```python
# ✅ Good: Explicit error handling
try:
    result = await execute_action(action)
    if not result.success:
        return ActionFailure(
            exception=ActionExecutionError(result.exception_message),
            stop_execution_on_failure=False
        )
except ElementNotFoundError as e:
    return ActionFailure(
        exception=e,
        stop_execution_on_failure=True
    )
```

---

## 🔄 Action Composition Patterns

### Sequential Action Chains
```python
class ActionChain:
    def __init__(self, actions: list[Action]):
        self.actions = self._validate_chain(actions)
    
    def _validate_chain(self, actions: list[Action]) -> list[Action]:
        """Validate that actions can be chained together"""
        for i, action in enumerate(actions):
            if isinstance(action, DecisiveAction) and i < len(actions) - 1:
                raise ValueError(
                    f"Decisive action {action.action_type} must be last in chain"
                )
        return actions
    
    async def execute(self, handler: ActionHandler) -> list[ActionResult]:
        results = []
        for action in self.actions:
            result = await handler.execute(action)
            results.append(result)
            
            # Stop chain on failure
            if not result.success and result.stop_execution_on_failure:
                break
                
        return results
```

### Conditional Action Execution
```python
class ConditionalAction:
    def __init__(
        self,
        condition: Callable[[ScrapedPage], bool],
        true_action: Action,
        false_action: Action | None = None
    ):
        self.condition = condition
        self.true_action = true_action
        self.false_action = false_action
    
    async def execute(
        self,
        handler: ActionHandler,
        scraped_page: ScrapedPage
    ) -> ActionResult:
        if self.condition(scraped_page):
            return await handler.execute(self.true_action)
        elif self.false_action:
            return await handler.execute(self.false_action)
        else:
            return ActionSuccess(data={"condition_not_met": True})
```

---

## 🎯 Element Targeting Best Practices

### Robust Element Selection
```python
class ElementSelector:
    @staticmethod
    def create_fallback_selectors(element_data: dict) -> list[str]:
        """Create multiple selector options for reliability"""
        selectors = []
        
        # Primary: Unique ID
        if element_data.get("id"):
            selectors.append(f"#{element_data['id']}")
        
        # Secondary: Data attributes
        if element_data.get("data-testid"):
            selectors.append(f"[data-testid='{element_data['data-testid']}']")
        
        # Tertiary: Class + text content
        if element_data.get("class") and element_data.get("text"):
            selectors.append(
                f".{element_data['class']}:has-text('{element_data['text']}')"
            )
        
        # Fallback: XPath
        if element_data.get("xpath"):
            selectors.append(f"xpath={element_data['xpath']}")
        
        return selectors
```

### Element Stability Verification
```python
async def verify_element_stability(
    locator: Locator,
    timeout_ms: int = 1000
) -> bool:
    """Verify element is stable before interaction"""
    try:
        # Wait for element to be stable
        await locator.wait_for(
            state="attached",
            timeout=timeout_ms
        )
        
        # Check if element is still visible and enabled
        is_visible = await locator.is_visible()
        is_enabled = await locator.is_enabled()
        
        return is_visible and is_enabled
    except TimeoutError:
        return False
```

---

## 📊 Performance Optimization Patterns

### Parallel Action Execution
```python
class ParallelActionExecutor:
    async def execute_parallel(
        self,
        actions: list[Action],
        handler: ActionHandler,
        max_concurrent: int = 3
    ) -> list[ActionResult]:
        """Execute non-dependent actions in parallel"""
        
        # Group actions by dependencies
        independent_actions = self._filter_independent_actions(actions)
        dependent_actions = self._filter_dependent_actions(actions)
        
        # Execute independent actions in parallel
        semaphore = asyncio.Semaphore(max_concurrent)
        
        async def execute_with_semaphore(action: Action) -> ActionResult:
            async with semaphore:
                return await handler.execute(action)
        
        parallel_results = await asyncio.gather(*[
            execute_with_semaphore(action) 
            for action in independent_actions
        ])
        
        # Execute dependent actions sequentially
        sequential_results = []
        for action in dependent_actions:
            result = await handler.execute(action)
            sequential_results.append(result)
        
        return parallel_results + sequential_results
```

### Smart Caching Strategy
```python
class ActionCacheManager:
    def __init__(self):
        self.cache_hit_rate = defaultdict(float)
        self.cache_staleness = defaultdict(int)
    
    async def should_use_cache(
        self,
        task: Task,
        scraped_page: ScrapedPage
    ) -> bool:
        """Intelligent cache usage decision"""
        
        # Check page similarity
        page_similarity = self._calculate_page_similarity(
            task.url, scraped_page
        )
        
        # Check cache hit rate for this URL pattern
        url_pattern = self._extract_url_pattern(task.url)
        hit_rate = self.cache_hit_rate[url_pattern]
        
        # Use cache if:
        # 1. Page similarity > 85%
        # 2. Cache hit rate > 70%
        # 3. Page not marked as dynamic
        return (
            page_similarity > 0.85 and
            hit_rate > 0.70 and
            not self._is_dynamic_page(scraped_page)
        )
```

---

## 🛡️ Error Handling Patterns

### Retry Strategy with Backoff
```python
class RetryStrategy:
    def __init__(
        self,
        max_retries: int = 3,
        base_delay: float = 1.0,
        max_delay: float = 10.0,
        backoff_factor: float = 2.0
    ):
        self.max_retries = max_retries
        self.base_delay = base_delay
        self.max_delay = max_delay
        self.backoff_factor = backoff_factor
    
    async def execute_with_retry(
        self,
        action: Action,
        handler: ActionHandler
    ) -> ActionResult:
        last_exception = None
        
        for attempt in range(self.max_retries + 1):
            try:
                result = await handler.execute(action)
                if result.success:
                    return result
                
                # Don't retry if action explicitly failed
                if result.stop_execution_on_failure:
                    return result
                    
            except Exception as e:
                last_exception = e
                
                # Don't retry on final attempt
                if attempt == self.max_retries:
                    break
                
                # Calculate delay with exponential backoff
                delay = min(
                    self.base_delay * (self.backoff_factor ** attempt),
                    self.max_delay
                )
                
                LOG.warning(
                    "Action failed, retrying",
                    action=action,
                    attempt=attempt + 1,
                    delay=delay,
                    exception=str(e)
                )
                
                await asyncio.sleep(delay)
        
        # All retries failed
        return ActionFailure(
            exception=last_exception or Exception("Max retries exceeded"),
            stop_execution_on_failure=True
        )
```

### Graceful Degradation
```python
class FallbackActionHandler:
    def __init__(self, primary_handler: ActionHandler):
        self.primary_handler = primary_handler
        self.fallback_strategies = {
            ActionType.CLICK: self._fallback_click,
            ActionType.INPUT_TEXT: self._fallback_input,
            ActionType.SELECT_OPTION: self._fallback_select
        }
    
    async def execute_with_fallback(self, action: Action) -> ActionResult:
        try:
            # Try primary execution
            result = await self.primary_handler.execute(action)
            if result.success:
                return result
        except Exception as e:
            LOG.warning("Primary action failed, trying fallback", 
                       action=action, exception=str(e))
        
        # Try fallback strategy
        fallback_func = self.fallback_strategies.get(action.action_type)
        if fallback_func:
            return await fallback_func(action)
        
        return ActionFailure(
            exception=Exception("No fallback strategy available"),
            stop_execution_on_failure=True
        )
    
    async def _fallback_click(self, action: ClickAction) -> ActionResult:
        """Fallback click using JavaScript"""
        try:
            # Use JavaScript click as fallback
            await self.primary_handler.page.evaluate(
                f"document.querySelector('[data-element-id=\"{action.element_id}\"]').click()"
            )
            return ActionSuccess(data={"fallback_method": "javascript_click"})
        except Exception as e:
            return ActionFailure(exception=e)
```

---

## 🔍 Debugging and Monitoring Patterns

### Action Execution Tracing
```python
class ActionTracer:
    def __init__(self):
        self.execution_trace = []
        self.performance_metrics = {}
    
    async def trace_execution(
        self,
        action: Action,
        handler: ActionHandler
    ) -> ActionResult:
        start_time = time.time()
        
        trace_entry = {
            "action_id": action.action_id,
            "action_type": action.action_type,
            "element_id": getattr(action, 'element_id', None),
            "start_time": start_time,
            "reasoning": action.reasoning
        }
        
        try:
            result = await handler.execute(action)
            
            trace_entry.update({
                "success": result.success,
                "duration_ms": (time.time() - start_time) * 1000,
                "result_data": result.data
            })
            
            if not result.success:
                trace_entry["error"] = result.exception_message
            
        except Exception as e:
            trace_entry.update({
                "success": False,
                "duration_ms": (time.time() - start_time) * 1000,
                "error": str(e)
            })
            result = ActionFailure(exception=e)
        
        self.execution_trace.append(trace_entry)
        return result
    
    def generate_execution_report(self) -> dict:
        """Generate comprehensive execution report"""
        total_actions = len(self.execution_trace)
        successful_actions = sum(1 for t in self.execution_trace if t["success"])
        
        action_type_stats = defaultdict(lambda: {"count": 0, "success": 0, "avg_duration": 0})
        
        for trace in self.execution_trace:
            action_type = trace["action_type"]
            stats = action_type_stats[action_type]
            stats["count"] += 1
            if trace["success"]:
                stats["success"] += 1
            stats["avg_duration"] += trace["duration_ms"]
        
        # Calculate averages
        for stats in action_type_stats.values():
            if stats["count"] > 0:
                stats["avg_duration"] /= stats["count"]
                stats["success_rate"] = stats["success"] / stats["count"]
        
        return {
            "summary": {
                "total_actions": total_actions,
                "successful_actions": successful_actions,
                "success_rate": successful_actions / total_actions if total_actions > 0 else 0,
                "total_duration_ms": sum(t["duration_ms"] for t in self.execution_trace)
            },
            "action_type_stats": dict(action_type_stats),
            "execution_trace": self.execution_trace
        }
```

---

## 🎯 Testing Patterns

### Action Unit Testing
```python
import pytest
from unittest.mock import AsyncMock, MagicMock

class TestActionExecution:
    @pytest.fixture
    async def mock_handler(self):
        handler = MagicMock()
        handler.browser_state = MagicMock()
        handler.scraped_page = MagicMock()
        return handler
    
    @pytest.mark.asyncio
    async def test_click_action_success(self, mock_handler):
        # Arrange
        action = ClickAction(
            element_id="test-button",
            reasoning="Test click action"
        )
        
        mock_element = AsyncMock()
        mock_handler.resolve_element = AsyncMock(return_value=mock_element)
        mock_element.click = AsyncMock()
        
        # Act
        result = await execute_click_action(action, mock_handler)
        
        # Assert
        assert result.success is True
        mock_element.click.assert_called_once()
        assert result.data["element_clicked"] is True
    
    @pytest.mark.asyncio
    async def test_input_text_action_validation(self, mock_handler):
        # Test input validation
        with pytest.raises(ValidationError):
            InputTextAction(
                element_id="",  # Invalid empty element_id
                text="test"
            )
        
        with pytest.raises(ValidationError):
            InputTextAction(
                element_id="input-field",
                text=""  # Invalid empty text
            )
```

### Integration Testing
```python
class TestActionIntegration:
    @pytest.mark.asyncio
    async def test_form_filling_workflow(self, browser_context):
        # Create test page
        page = await browser_context.new_page()
        await page.goto("data:text/html,<form><input id='name'><input id='email'><button id='submit'>Submit</button></form>")
        
        # Create actions
        actions = [
            InputTextAction(element_id="name", text="John Doe"),
            InputTextAction(element_id="email", text="john@example.com"),
            ClickAction(element_id="submit")
        ]
        
        # Execute actions
        handler = ActionHandler(browser_state=BrowserState(page=page))
        results = []
        
        for action in actions:
            result = await handler.execute(action)
            results.append(result)
            assert result.success, f"Action {action.action_type} failed: {result.exception_message}"
        
        # Verify form submission
        assert len(results) == 3
        assert all(r.success for r in results)
```

---

## 📝 Documentation Patterns

### Self-Documenting Actions
```python
class DocumentedAction(Action):
    """Base class for self-documenting actions"""
    
    def get_action_description(self) -> str:
        """Generate human-readable action description"""
        return f"{self.action_type.title()} action: {self.reasoning}"
    
    def get_execution_context(self) -> dict:
        """Get context information for debugging"""
        return {
            "action_type": self.action_type,
            "element_id": getattr(self, 'element_id', None),
            "confidence": self.confidence_float,
            "reasoning": self.reasoning,
            "step_order": self.step_order,
            "action_order": self.action_order
        }
    
    def validate_preconditions(self, scraped_page: ScrapedPage) -> list[str]:
        """Validate that action can be executed"""
        issues = []
        
        if hasattr(self, 'element_id'):
            if self.element_id not in scraped_page.id_to_element_dict:
                issues.append(f"Element {self.element_id} not found on page")
        
        return issues
```

### Action Logging Standards
```python
class ActionLogger:
    @staticmethod
    def log_action_start(action: Action):
        LOG.info(
            "Starting action execution",
            action_type=action.action_type,
            action_id=action.action_id,
            element_id=getattr(action, 'element_id', None),
            reasoning=action.reasoning,
            confidence=action.confidence_float
        )
    
    @staticmethod
    def log_action_result(action: Action, result: ActionResult):
        if result.success:
            LOG.info(
                "Action completed successfully",
                action_type=action.action_type,
                action_id=action.action_id,
                duration_ms=result.data.get('execution_time_ms'),
                result_data=result.data
            )
        else:
            LOG.error(
                "Action failed",
                action_type=action.action_type,
                action_id=action.action_id,
                exception_type=result.exception_type,
                exception_message=result.exception_message,
                stop_execution=result.stop_execution_on_failure
            )
```

---

## 🚀 Advanced Patterns

### Dynamic Action Generation
```python
class ActionFactory:
    @staticmethod
    def create_form_filling_actions(
        form_data: dict,
        element_map: dict
    ) -> list[Action]:
        """Generate actions to fill a form based on data"""
        actions = []
        
        for field_name, value in form_data.items():
            element_id = element_map.get(field_name)
            if not element_id:
                continue
            
            if isinstance(value, str):
                actions.append(InputTextAction(
                    element_id=element_id,
                    text=value,
                    reasoning=f"Fill {field_name} field with provided value"
                ))
            elif isinstance(value, dict) and 'option' in value:
                actions.append(SelectOptionAction(
                    element_id=element_id,
                    option=SelectOption(**value['option']),
                    reasoning=f"Select option for {field_name} field"
                ))
        
        return actions
    
    @staticmethod
    def create_navigation_actions(
        target_url: str,
        current_url: str
    ) -> list[Action]:
        """Generate actions to navigate to target URL"""
        actions = []
        
        if target_url != current_url:
            # Add navigation logic based on URL patterns
            if "login" in target_url and "login" not in current_url:
                actions.append(ClickAction(
                    element_id="login-link",
                    reasoning="Navigate to login page"
                ))
        
        return actions
```

### Adaptive Action Execution
```python
class AdaptiveActionExecutor:
    def __init__(self):
        self.success_patterns = defaultdict(list)
        self.failure_patterns = defaultdict(list)
    
    async def execute_with_adaptation(
        self,
        action: Action,
        handler: ActionHandler,
        scraped_page: ScrapedPage
    ) -> ActionResult:
        """Execute action with adaptive behavior based on past patterns"""
        
        # Analyze page context
        page_context = self._analyze_page_context(scraped_page)
        
        # Check for known successful patterns
        successful_strategy = self._find_successful_strategy(
            action.action_type, page_context
        )
        
        if successful_strategy:
            # Use proven successful approach
            return await self._execute_with_strategy(
                action, handler, successful_strategy
            )
        
        # Try standard execution
        result = await handler.execute(action)
        
        # Learn from result
        if result.success:
            self.success_patterns[action.action_type].append(page_context)
        else:
            self.failure_patterns[action.action_type].append(page_context)
        
        return result
```