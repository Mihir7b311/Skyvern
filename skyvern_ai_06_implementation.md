# 💻 Implementation Examples
## Code Samples, Best Practices & Troubleshooting

---

## 🎯 Overview

This section provides practical implementation examples, configuration samples, and best practices for integrating Skyvern's AI & LLM components into your automation workflows.

---

## 🔧 Basic Implementation Setup

### **1. LLM Provider Configuration**

```python
# config/llm_setup.py
from skyvern.forge.sdk.api.llm.config_registry import LLMConfigRegistry
from skyvern.forge.sdk.api.llm.models import LLMConfig, LiteLLMParams

def setup_llm_providers():
    """Configure multiple LLM providers with fallback"""
    
    # Primary provider: OpenAI GPT-4o
    primary_config = LLMConfig(
        model_name="gpt-4o",
        required_env_vars=["OPENAI_API_KEY"],
        supports_vision=True,
        add_assistant_prefix=False,
        max_tokens=4000,
        temperature=0.1,
        litellm_params=LiteLLMParams(
            api_key=settings.OPENAI_API_KEY,
            api_base="https://api.openai.com/v1"
        )
    )
    
    # Secondary provider: Claude-3.5 Sonnet
    secondary_config = LLMConfig(
        model_name="claude-3-5-sonnet-20241022",
        required_env_vars=["ANTHROPIC_API_KEY"],
        supports_vision=True,
        add_assistant_prefix=True,
        max_tokens=4000,
        temperature=0.1,
        litellm_params=LiteLLMParams(
            api_key=settings.ANTHROPIC_API_KEY,
            api_base="https://api.anthropic.com"
        )
    )
    
    # Cost-effective provider: GPT-4o-mini
    budget_config = LLMConfig(
        model_name="gpt-4o-mini",
        required_env_vars=["OPENAI_API_KEY"],
        supports_vision=True,
        add_assistant_prefix=False,
        max_tokens=2000,
        temperature=0.2,
        litellm_params=LiteLLMParams(
            api_key=settings.OPENAI_API_KEY,
            api_base="https://api.openai.com/v1"
        )
    )
    
    # Register configurations
    LLMConfigRegistry.register_config("primary-llm", primary_config)
    LLMConfigRegistry.register_config("secondary-llm", secondary_config)
    LLMConfigRegistry.register_config("budget-llm", budget_config)

# Initialize during app startup
setup_llm_providers()
```

### **2. Environment Configuration**

```bash
# .env file
# Primary LLM Configuration
LLM_KEY=primary-llm
SECONDARY_LLM_KEY=secondary-llm

# API Keys
OPENAI_API_KEY=sk-your-openai-api-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
OPENROUTER_API_KEY=sk-or-your-openrouter-key

# LLM Settings
LLM_CONFIG_MAX_TOKENS=4000
LLM_CONFIG_TEMPERATURE=0.1
LLM_CONFIG_SUPPORT_VISION=true
LLM_CONFIG_ADD_ASSISTANT_PREFIX=false

# Performance Settings
PROMPT_CACHE_WINDOW_HOURS=24
LLM_REQUEST_TIMEOUT=30
MAX_CONCURRENT_LLM_REQUESTS=10

# Monitoring
ENABLE_LLM_METRICS=true
LOG_LLM_REQUESTS=true
LOG_LLM_RESPONSES=false  # Set to true for debugging
```

---

## 📝 Prompt Template Examples

### **3. Custom Action Extraction Template**

```jinja2
{# templates/custom-action-extraction.j2 #}
You are an expert web automation assistant. Analyze the provided screenshot and DOM elements to determine the best action to achieve the user's goal.

**Current Goal:** {{ user_goal }}
**Current URL:** {{ current_url }}
**Step {{ step_number }} of {{ max_steps }}**

{% if error_context %}
**Previous Error:** {{ error_context.message }}
**Failed Action:** {{ error_context.action_type }}
Please try a different approach.
{% endif %}

**Available Elements:**
{% for element in elements %}
- ID: `{{ element.id }}` | Type: `{{ element.tag }}` | Text: "{{ element.text[:50] }}" | Visible: {{ element.is_visible }}
{% endfor %}

**Rules:**
1. Only interact with visible elements (is_visible: true)
2. Prefer elements with clear, relevant text or labels
3. Use INPUT_TEXT for form fields, CLICK for buttons/links
4. Include confidence_float (0.0-1.0) for each action
5. Provide clear reasoning for your choice

**Response Format:**
```json
{
    "actions": [
        {
            "action_type": "ACTION_TYPE",
            "element_id": "element_id_here",
            "text": "text_to_input",  // Only for INPUT_TEXT
            "reasoning": "Why this action helps achieve the goal",
            "confidence_float": 0.85
        }
    ]
}
```

**Your analysis and action:**
```

### **4. Data Extraction Template**

```jinja2
{# templates/data-extraction.j2 #}
Extract structured data from the current page according to the specified schema.

**Extraction Goal:** {{ data_extraction_goal }}
**Target Schema:**
{{ data_schema | tojson(indent=2) }}

**Page Content Analysis:**
{% if screenshots %}
- Visual content available for analysis
{% endif %}
{% if elements %}
- {{ elements|length }} DOM elements detected
{% endif %}

**Extraction Instructions:**
1. Analyze visible text content and form values
2. Match data to schema fields exactly
3. Use null for missing/unavailable data
4. Ensure data types match schema requirements
5. Include confidence scores for extracted values

**Response Format:**
```json
{
    "extracted_data": {
        {% for field_name, field_spec in data_schema.properties.items() %}
        "{{ field_name }}": null,  // {{ field_spec.description }}
        {% endfor %}
    },
    "extraction_confidence": 0.0,
    "missing_fields": [],
    "extraction_notes": "Any relevant observations"
}
```

Extract the data now:
```

---

## 🎮 Response Processing Examples

### **5. Custom Response Parser**

```python
# parsers/custom_response_parser.py
from typing import Any, Dict, List
import json
import re
from skyvern.webeye.actions.actions import Action, ClickAction, InputTextAction, NullAction
from skyvern.webeye.actions.action_types import ActionType

class CustomResponseParser:
    """Custom response parser with enhanced error recovery"""
    
    def __init__(self):
        self.recovery_strategies = [
            self._try_direct_json,
            self._try_markdown_extraction,
            self._try_regex_extraction,
            self._try_text_parsing,
            self._create_fallback_action
        ]
    
    async def parse_response(
        self, 
        response_content: str,
        context: Dict[str, Any]
    ) -> List[Action]:
        """Parse LLM response with multiple fallback strategies"""
        
        for strategy_idx, strategy in enumerate(self.recovery_strategies):
            try:
                actions = await strategy(response_content, context)
                if actions:
                    LOG.info(f"Parsing successful with strategy {strategy_idx}")
                    return actions
            except Exception as e:
                LOG.warning(f"Strategy {strategy_idx} failed: {e}")
                continue
        
        LOG.error("All parsing strategies failed")
        return [NullAction(reasoning="Failed to parse LLM response")]
    
    async def _try_direct_json(self, content: str, context: Dict[str, Any]) -> List[Action]:
        """Attempt direct JSON parsing"""
        parsed = json.loads(content)
        return self._convert_to_actions(parsed.get("actions", []), context)
    
    async def _try_markdown_extraction(self, content: str, context: Dict[str, Any]) -> List[Action]:
        """Extract JSON from markdown code blocks"""
        json_pattern = r'```(?:json)?\s*(.*?)\s*```'
        match = re.search(json_pattern, content, re.DOTALL)
        if match:
            json_content = match.group(1)
            parsed = json.loads(json_content)
            return self._convert_to_actions(parsed.get("actions", []), context)
        return []
    
    async def _try_regex_extraction(self, content: str, context: Dict[str, Any]) -> List[Action]:
        """Extract actions using regex patterns"""
        actions = []
        
        # Pattern for action extraction
        action_pattern = r'action_type["\']\s*:\s*["\'](\w+)["\'].*?element_id["\']\s*:\s*["\']([^"\']+)["\']'
        matches = re.finditer(action_pattern, content, re.IGNORECASE | re.DOTALL)
        
        for match in matches:
            action_type = match.group(1).upper()
            element_id = match.group(2)
            
            if action_type == "CLICK":
                actions.append(ClickAction(
                    element_id=element_id,
                    reasoning="Extracted from regex parsing"
                ))
            elif action_type == "INPUT_TEXT":
                # Try to extract text value
                text_pattern = f'element_id["\']\s*:\s*["\']{{element_id}}["\'].*?text["\']\s*:\s*["\']([^"\']+)["\']'
                text_match = re.search(text_pattern.format(element_id=element_id), content)
                text_value = text_match.group(1) if text_match else ""
                
                actions.append(InputTextAction(
                    element_id=element_id,
                    text=text_value,
                    reasoning="Extracted from regex parsing"
                ))
        
        return actions
    
    async def _try_text_parsing(self, content: str, context: Dict[str, Any]) -> List[Action]:
        """Parse actions from natural language description"""
        # Simple keyword-based parsing
        content_lower = content.lower()
        
        if "click" in content_lower and "button" in content_lower:
            # Look for button-like elements in context
            for element in context.get("elements", []):
                if element.get("tag") == "button" or "btn" in element.get("id", ""):
                    return [ClickAction(
                        element_id=element["id"],
                        reasoning="Inferred click action from text"
                    )]
        
        return []
    
    async def _create_fallback_action(self, content: str, context: Dict[str, Any]) -> List[Action]:
        """Create a safe fallback action"""
        return [NullAction(
            reasoning=f"Unable to parse response: {content[:100]}..."
        )]
    
    def _convert_to_actions(self, action_dicts: List[Dict], context: Dict[str, Any]) -> List[Action]:
        """Convert action dictionaries to Action objects"""
        actions = []
        
        for action_dict in action_dicts:
            try:
                action_type = ActionType[action_dict["action_type"].upper()]
                
                base_params = {
                    "element_id": action_dict.get("element_id"),
                    "reasoning": action_dict.get("reasoning", ""),
                    "confidence_float": action_dict.get("confidence_float", 0.5)
                }
                
                if action_type == ActionType.CLICK:
                    actions.append(ClickAction(**base_params))
                elif action_type == ActionType.INPUT_TEXT:
                    actions.append(InputTextAction(
                        **base_params,
                        text=action_dict.get("text", "")
                    ))
                # Add more action types as needed
                
            except Exception as e:
                LOG.warning(f"Failed to convert action: {e}")
                continue
        
        return actions
```

---

## 🎯 Complete Workflow Implementation

### **6. End-to-End Task Automation**

```python
# workflows/task_automation.py
from typing import Any, Dict, List, Optional
import asyncio
from skyvern.forge.sdk.schemas.tasks import Task, TaskStatus
from skyvern.forge.sdk.models import Step, StepStatus
from skyvern.webeye.browser_factory import BrowserState
from skyvern.forge.sdk.api.llm.api_handler_factory import LLMAPIHandlerFactory

class TaskAutomationWorkflow:
    """Complete task automation workflow with AI decision making"""
    
    def __init__(self):
        self.llm_handler = LLMAPIHandlerFactory.get_llm_api_handler("primary-llm")
        self.response_parser = CustomResponseParser()
        self.max_steps = 20
        self.step_timeout = 60.0
    
    async def execute_task(
        self,
        task: Task,
        browser_state: BrowserState,
        initial_url: str
    ) -> Dict[str, Any]:
        """Execute complete automation task"""
        
        LOG.info(f"Starting task execution: {task.task_id}")
        
        # Initialize task context
        task_context = {
            "goal": task.navigation_goal,
            "current_url": initial_url,
            "step_history": [],
            "extracted_data": {},
            "errors": []
        }
        
        try:
            # Navigate to initial URL
            await self._navigate_to_url(browser_state, initial_url)
            
            # Execute steps until goal achieved or max steps reached
            for step_number in range(1, self.max_steps + 1):
                step_result = await self._execute_single_step(
                    task=task,
                    browser_state=browser_state,
                    step_number=step_number,
                    context=task_context
                )
                
                task_context["step_history"].append(step_result)
                
                # Check if task is complete
                if step_result["status"] == "completed":
                    LOG.info(f"Task completed successfully in {step_number} steps")
                    break
                elif step_result["status"] == "failed":
                    LOG.error(f"Task failed at step {step_number}")
                    break
                
                # Brief pause between steps
                await asyncio.sleep(1.0)
            
            # Final data extraction if specified
            if task.data_extraction_goal:
                final_data = await self._extract_final_data(
                    task=task,
                    browser_state=browser_state,
                    context=task_context
                )
                task_context["extracted_data"] = final_data
            
            return {
                "status": "success",
                "steps_completed": len(task_context["step_history"]),
                "extracted_data": task_context["extracted_data"],
                "final_url": await self._get_current_url(browser_state)
            }
            
        except Exception as e:
            LOG.exception(f"Task execution failed: {e}")
            return {
                "status": "error",
                "error": str(e),
                "steps_completed": len(task_context["step_history"])
            }
    
    async def _execute_single_step(
        self,
        task: Task,
        browser_state: BrowserState,
        step_number: int,
        context: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Execute a single automation step"""
        
        step_start_time = asyncio.get_event_loop().time()
        
        try:
            # 1. Capture current page state
            page_state = await self._capture_page_state(browser_state)
            
    async def _execute_single_step(
        self,
        task: Task,
        browser_state: BrowserState,
        step_number: int,
        context: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Execute a single automation step"""
        
        step_start_time = asyncio.get_event_loop().time()
        
        try:
            # 1. Capture current page state
            page_state = await self._capture_page_state(browser_state)
            
            # 2. Generate context-aware prompt
            prompt = await self._build_step_prompt(
                goal=context["goal"],
                current_url=context["current_url"],
                step_number=step_number,
                max_steps=self.max_steps,
                page_state=page_state,
                step_history=context["step_history"][-3:]  # Last 3 steps for context
            )
            
            # 3. Get AI response
            llm_response = await asyncio.wait_for(
                self.llm_handler(
                    prompt=prompt,
                    prompt_name="task-step-execution",
                    screenshots=page_state.get("screenshots", [])
                ),
                timeout=self.step_timeout
            )
            
            # 4. Parse actions from response
            actions = await self.response_parser.parse_response(
                response_content=str(llm_response),
                context={
                    "elements": page_state.get("elements", []),
                    "goal": context["goal"]
                }
            )
            
            # 5. Execute actions
            execution_results = []
            for action in actions:
                result = await self._execute_action(browser_state, action)
                execution_results.append(result)
                
                # Stop if action failed critically
                if result["status"] == "critical_failure":
                    break
            
            # 6. Evaluate step success
            step_success = self._evaluate_step_success(actions, execution_results)
            
            step_duration = asyncio.get_event_loop().time() - step_start_time
            
            return {
                "step_number": step_number,
                "status": "completed" if step_success else "failed",
                "actions": [self._action_to_dict(a) for a in actions],
                "results": execution_results,
                "duration": step_duration,
                "page_url": await self._get_current_url(browser_state)
            }
            
        except asyncio.TimeoutError:
            return {
                "step_number": step_number,
                "status": "timeout",
                "error": f"Step timed out after {self.step_timeout}s"
            }
        except Exception as e:
            LOG.exception(f"Step {step_number} execution failed")
            return {
                "step_number": step_number,
                "status": "failed",
                "error": str(e)
            }
    
    async def _build_step_prompt(
        self,
        goal: str,
        current_url: str,
        step_number: int,
        max_steps: int,
        page_state: Dict[str, Any],
        step_history: List[Dict[str, Any]]
    ) -> str:
        """Build context-aware prompt for current step"""
        
        # Extract key elements for prompt
        visible_elements = [
            elem for elem in page_state.get("elements", [])
            if elem.get("is_visible", False)
        ][:10]  # Limit to top 10 visible elements
        
        # Build step history context
        history_context = ""
        if step_history:
            history_context = "**Previous Steps:**\n"
            for hist_step in step_history:
                history_context += f"Step {hist_step['step_number']}: {hist_step.get('summary', 'Action taken')}\n"
        
        # Build error context if any
        error_context = ""
        recent_errors = [
            step for step in step_history 
            if step.get("status") == "failed"
        ]
        if recent_errors:
            last_error = recent_errors[-1]
            error_context = f"**Previous Error:** {last_error.get('error', 'Unknown error')}\n"
        
        prompt = f"""
You are automating a web browser to achieve this goal: {goal}

**Current Status:**
- URL: {current_url}
- Step: {step_number} of {max_steps}

{history_context}
{error_context}

**Available Elements:**
"""
        
        for i, elem in enumerate(visible_elements):
            prompt += f"{i+1}. ID: '{elem.get('id', 'N/A')}' | Type: {elem.get('tag', 'unknown')} | Text: \"{elem.get('text', '')[:50]}\"\n"
        
        prompt += """
**Instructions:**
1. Analyze the current page and determine the best action to progress toward the goal
2. Choose ONE action that moves closest to achieving the goal
3. Prefer visible, interactive elements
4. Provide confidence score (0.0-1.0) for your action choice

**Response Format:**
```json
{
    "actions": [
        {
            "action_type": "CLICK|INPUT_TEXT|SELECT_OPTION",
            "element_id": "element_id_here",
            "text": "text_for_input_actions",
            "reasoning": "Why this action helps achieve the goal",
            "confidence_float": 0.85
        }
    ]
}
```

Your action:
"""
        return prompt
    
    async def _execute_action(
        self, 
        browser_state: BrowserState, 
        action: Action
    ) -> Dict[str, Any]:
        """Execute a single action and return result"""
        
        try:
            action_start_time = asyncio.get_event_loop().time()
            
            # Execute action based on type
            if isinstance(action, ClickAction):
                await self._perform_click(browser_state, action)
                result_status = "success"
                result_message = f"Clicked element {action.element_id}"
                
            elif isinstance(action, InputTextAction):
                await self._perform_input_text(browser_state, action)
                result_status = "success"
                result_message = f"Entered text into {action.element_id}"
                
            else:
                result_status = "skipped"
                result_message = f"Action type {type(action).__name__} not implemented"
            
            execution_time = asyncio.get_event_loop().time() - action_start_time
            
            # Wait for page to stabilize
            await asyncio.sleep(0.5)
            
            return {
                "action_type": type(action).__name__,
                "element_id": action.element_id,
                "status": result_status,
                "message": result_message,
                "execution_time": execution_time
            }
            
        except Exception as e:
            LOG.error(f"Action execution failed: {e}")
            return {
                "action_type": type(action).__name__,
                "element_id": action.element_id,
                "status": "critical_failure",
                "error": str(e)
            }
    
    async def _perform_click(self, browser_state: BrowserState, action: ClickAction):
        """Perform click action on element"""
        page = browser_state.page
        if action.element_id:
            # Try multiple selector strategies
            selectors = [
                f"[data-skyvern-id='{action.element_id}']",
                f"#{action.element_id}",
                f"[id='{action.element_id}']"
            ]
            
            for selector in selectors:
                try:
                    element = await page.wait_for_selector(selector, timeout=5000)
                    if element:
                        await element.click()
                        return
                except Exception:
                    continue
            
            raise Exception(f"Element {action.element_id} not found")
    
    async def _perform_input_text(self, browser_state: BrowserState, action: InputTextAction):
        """Perform text input action"""
        page = browser_state.page
        if action.element_id and action.text:
            selectors = [
                f"[data-skyvern-id='{action.element_id}']",
                f"#{action.element_id}",
                f"[id='{action.element_id}']"
            ]
            
            for selector in selectors:
                try:
                    element = await page.wait_for_selector(selector, timeout=5000)
                    if element:
                        await element.clear()
                        await element.fill(action.text)
                        return
                except Exception:
                    continue
            
            raise Exception(f"Input element {action.element_id} not found")
```

---

## 🔧 Configuration Best Practices

### **7. Production Configuration**

```python
# config/production.py
import os
from dataclasses import dataclass

@dataclass
class ProductionConfig:
    """Production-ready configuration for Skyvern AI"""
    
    # LLM Configuration
    PRIMARY_LLM_KEY: str = "gpt-4o"
    SECONDARY_LLM_KEY: str = "claude-3-5-sonnet-20241022"
    BUDGET_LLM_KEY: str = "gpt-4o-mini"
    
    # Performance Settings
    MAX_CONCURRENT_TASKS: int = 50
    LLM_REQUEST_TIMEOUT: int = 30
    MAX_TOKENS_PER_REQUEST: int = 4000
    
    # Retry Configuration
    MAX_RETRY_ATTEMPTS: int = 3
    RETRY_BACKOFF_FACTOR: float = 2.0
    RETRY_JITTER: bool = True
    
    # Caching
    ENABLE_RESPONSE_CACHE: bool = True
    CACHE_TTL_SECONDS: int = 3600
    MAX_CACHE_SIZE: int = 1000
    
    # Monitoring
    ENABLE_METRICS: bool = True
    METRICS_EXPORT_INTERVAL: int = 60
    LOG_LEVEL: str = "INFO"
    
    # Security
    ENABLE_REQUEST_VALIDATION: bool = True
    MAX_PROMPT_LENGTH: int = 50000
    SANITIZE_OUTPUTS: bool = True
    
    @classmethod
    def from_env(cls) -> "ProductionConfig":
        """Load configuration from environment variables"""
        return cls(
            PRIMARY_LLM_KEY=os.getenv("PRIMARY_LLM_KEY", cls.PRIMARY_LLM_KEY),
            SECONDARY_LLM_KEY=os.getenv("SECONDARY_LLM_KEY", cls.SECONDARY_LLM_KEY),
            MAX_CONCURRENT_TASKS=int(os.getenv("MAX_CONCURRENT_TASKS", cls.MAX_CONCURRENT_TASKS)),
            LLM_REQUEST_TIMEOUT=int(os.getenv("LLM_REQUEST_TIMEOUT", cls.LLM_REQUEST_TIMEOUT)),
            # ... other environment mappings
        )

# Usage
config = ProductionConfig.from_env()
```

### **8. Monitoring & Observability**

```python
# monitoring/llm_monitor.py
import time
import asyncio
from typing import Dict, Any, List
from dataclasses import dataclass, field
from collections import defaultdict, deque

@dataclass
class LLMMetrics:
    """LLM performance metrics"""
    total_requests: int = 0
    successful_requests: int = 0
    failed_requests: int = 0
    total_tokens_used: int = 0
    total_cost: float = 0.0
    avg_response_time: float = 0.0
    error_rates: Dict[str, int] = field(default_factory=lambda: defaultdict(int))
    recent_response_times: deque = field(default_factory=lambda: deque(maxlen=100))

class LLMMonitor:
    """Monitor LLM performance and costs"""
    
    def __init__(self):
        self.metrics = defaultdict(LLMMetrics)
        self.alert_thresholds = {
            "error_rate": 0.1,  # 10% error rate
            "avg_response_time": 10.0,  # 10 seconds
            "cost_per_hour": 100.0  # $100/hour
        }
        self.alerts_sent = set()
    
    async def record_request(
        self,
        provider: str,
        model: str,
        success: bool,
        response_time: float,
        tokens_used: int = 0,
        cost: float = 0.0,
        error_type: str = None
    ):
        """Record LLM request metrics"""
        
        metric_key = f"{provider}:{model}"
        metrics = self.metrics[metric_key]
        
        # Update counters
        metrics.total_requests += 1
        if success:
            metrics.successful_requests += 1
        else:
            metrics.failed_requests += 1
            if error_type:
                metrics.error_rates[error_type] += 1
        
        # Update usage metrics
        metrics.total_tokens_used += tokens_used
        metrics.total_cost += cost
        metrics.recent_response_times.append(response_time)
        
        # Calculate rolling average response time
        if metrics.recent_response_times:
            metrics.avg_response_time = sum(metrics.recent_response_times) / len(metrics.recent_response_times)
        
        # Check for alerts
        await self._check_alerts(metric_key, metrics)
    
    async def _check_alerts(self, metric_key: str, metrics: LLMMetrics):
        """Check if metrics exceed alert thresholds"""
        
        # Error rate alert
        if metrics.total_requests > 10:  # Need minimum requests
            error_rate = metrics.failed_requests / metrics.total_requests
            if error_rate > self.alert_thresholds["error_rate"]:
                alert_key = f"{metric_key}:error_rate"
                if alert_key not in self.alerts_sent:
                    await self._send_alert(
                        "HIGH_ERROR_RATE",
                        f"Error rate for {metric_key}: {error_rate:.2%}"
                    )
                    self.alerts_sent.add(alert_key)
        
        # Response time alert
        if metrics.avg_response_time > self.alert_thresholds["avg_response_time"]:
            alert_key = f"{metric_key}:response_time"
            if alert_key not in self.alerts_sent:
                await self._send_alert(
                    "SLOW_RESPONSE",
                    f"Slow response time for {metric_key}: {metrics.avg_response_time:.2f}s"
                )
                self.alerts_sent.add(alert_key)
    
    async def _send_alert(self, alert_type: str, message: str):
        """Send alert notification"""
        LOG.warning(f"ALERT [{alert_type}]: {message}")
        # Implement actual alerting (email, Slack, PagerDuty, etc.)
    
    def get_summary_report(self) -> Dict[str, Any]:
        """Generate summary report of all metrics"""
        
        report = {
            "timestamp": time.time(),
            "providers": {}
        }
        
        total_requests = 0
        total_cost = 0.0
        
        for metric_key, metrics in self.metrics.items():
            provider, model = metric_key.split(":", 1)
            
            if provider not in report["providers"]:
                report["providers"][provider] = {}
            
            report["providers"][provider][model] = {
                "total_requests": metrics.total_requests,
                "success_rate": (
                    metrics.successful_requests / metrics.total_requests
                    if metrics.total_requests > 0 else 0
                ),
                "avg_response_time": metrics.avg_response_time,
                "total_tokens": metrics.total_tokens_used,
                "total_cost": metrics.total_cost,
                "error_breakdown": dict(metrics.error_rates)
            }
            
            total_requests += metrics.total_requests
            total_cost += metrics.total_cost
        
        report["summary"] = {
            "total_requests": total_requests,
            "total_cost": total_cost,
            "cost_per_request": total_cost / total_requests if total_requests > 0 else 0
        }
        
        return report

# Global monitor instance
llm_monitor = LLMMonitor()

# Usage in LLM handler
async def monitored_llm_call(provider: str, model: str, prompt: str) -> str:
    """LLM call with monitoring"""
    start_time = time.time()
    
    try:
        # Make LLM call
        response = await make_llm_call(provider, model, prompt)
        
        # Record success
        await llm_monitor.record_request(
            provider=provider,
            model=model,
            success=True,
            response_time=time.time() - start_time,
            tokens_used=calculate_tokens(prompt, response),
            cost=calculate_cost(provider, model, tokens_used)
        )
        
        return response
        
    except Exception as e:
        # Record failure
        await llm_monitor.record_request(
            provider=provider,
            model=model,
            success=False,
            response_time=time.time() - start_time,
            error_type=type(e).__name__
        )
        raise
```

---

## 🐛 Debugging & Troubleshooting

### **9. Debug Tools**

```python
# debug/llm_debugger.py
from typing import Any, Dict, List, Optional
import json
import asyncio
from pathlib import Path

class LLMDebugger:
    """Comprehensive debugging tools for LLM integration"""
    
    def __init__(self, debug_dir: Path = Path("debug_logs")):
        self.debug_dir = debug_dir
        self.debug_dir.mkdir(exist_ok=True)
        self.session_id = int(time.time())
    
    async def debug_complete_flow(
        self,
        task_id: str,
        user_goal: str,
        initial_url: str
    ) -> Dict[str, Any]:
        """Debug complete AI flow from start to finish"""
        
        debug_session = {
            "session_id": self.session_id,
            "task_id": task_id,
            "user_goal": user_goal,
            "initial_url": initial_url,
            "steps": [],
            "summary": {}
        }
        
        LOG.info(f"Starting debug session {self.session_id} for task {task_id}")
        
        try:
            # Simulate complete workflow with detailed logging
            for step_num in range(1, 6):  # Debug first 5 steps
                step_debug = await self._debug_single_step(
                    task_id=task_id,
                    step_number=step_num,
                    user_goal=user_goal,
                    session=debug_session
                )
                debug_session["steps"].append(step_debug)
                
                # Stop if critical error
                if step_debug.get("critical_error"):
                    break
            
            # Generate summary
            debug_session["summary"] = self._generate_debug_summary(debug_session)
            
            # Save debug session
            await self._save_debug_session(debug_session)
            
            return debug_session
            
        except Exception as e:
            LOG.exception(f"Debug session {self.session_id} failed")
            debug_session["fatal_error"] = str(e)
            return debug_session
    
    async def _debug_single_step(
        self,
        task_id: str,
        step_number: int,
        user_goal: str,
        session: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Debug a single automation step in detail"""
        
        step_debug = {
            "step_number": step_number,
            "timestamp": time.time(),
            "phases": {}
        }
        
        try:
            # Phase 1: Context Building
            step_debug["phases"]["context_building"] = await self._debug_context_building(
                task_id, step_number, user_goal
            )
            
            # Phase 2: Prompt Generation
            step_debug["phases"]["prompt_generation"] = await self._debug_prompt_generation(
                user_goal, step_number
            )
            
            # Phase 3: LLM Processing
            step_debug["phases"]["llm_processing"] = await self._debug_llm_processing(
                step_debug["phases"]["prompt_generation"]["final_prompt"]
            )
            
            # Phase 4: Response Parsing
            step_debug["phases"]["response_parsing"] = await self._debug_response_parsing(
                step_debug["phases"]["llm_processing"]["raw_response"]
            )
            
            # Phase 5: Action Execution (simulated)
            step_debug["phases"]["action_execution"] = await self._debug_action_execution(
                step_debug["phases"]["response_parsing"]["parsed_actions"]
            )
            
            return step_debug
            
        except Exception as e:
            step_debug["critical_error"] = str(e)
            LOG.exception(f"Step {step_number} debug failed")
            return step_debug
    
    async def _debug_context_building(
        self, 
        task_id: str, 
        step_number: int, 
        user_goal: str
    ) -> Dict[str, Any]:
        """Debug context building phase"""
        
        return {
            "phase": "context_building",
            "inputs": {
                "task_id": task_id,
                "step_number": step_number,
                "user_goal": user_goal
            },
            "mock_page_elements": [
                {"id": "email-input", "type": "input", "visible": True},
                {"id": "password-input", "type": "input", "visible": True},
                {"id": "login-button", "type": "button", "visible": True}
            ],
            "context_quality_score": 0.85,
            "warnings": []
        }
    
    async def _debug_prompt_generation(
        self, 
        user_goal: str, 
        step_number: int
    ) -> Dict[str, Any]:
        """Debug prompt generation phase"""
        
        mock_prompt = f"""
Goal: {user_goal}
Step: {step_number}
Elements: [mock elements]
Action: Choose best action to progress toward goal
"""
        
        return {
            "phase": "prompt_generation",
            "template_used": "action-extraction",
            "parameters": {
                "user_goal": user_goal,
                "step_number": step_number,
                "elements_count": 3
            },
            "final_prompt": mock_prompt,
            "prompt_length": len(mock_prompt),
            "estimated_tokens": len(mock_prompt.split()) * 1.3,
            "warnings": []
        }
    
    async def _debug_llm_processing(self, prompt: str) -> Dict[str, Any]:
        """Debug LLM processing phase"""
        
        # Simulate LLM call
        await asyncio.sleep(0.1)  # Simulate network delay
        
        mock_response = {
            "actions": [
                {
                    "action_type": "INPUT_TEXT",
                    "element_id": "email-input",
                    "text": "user@example.com",
                    "reasoning": "Enter email to proceed with login",
                    "confidence_float": 0.9
                }
            ]
        }
        
        return {
            "phase": "llm_processing",
            "provider": "openai",
            "model": "gpt-4o",
            "request_tokens": 150,
            "response_tokens": 50,
            "processing_time": 2.3,
            "raw_response": json.dumps(mock_response),
            "cost_estimate": 0.003,
            "warnings": []
        }
    
    async def _debug_response_parsing(self, raw_response: str) -> Dict[str, Any]:
        """Debug response parsing phase"""
        
        try:
            parsed = json.loads(raw_response)
            parsing_success = True
            parsing_error = None
        except Exception as e:
            parsed = None
            parsing_success = False
            parsing_error = str(e)
        
        return {
            "phase": "response_parsing",
            "parsing_strategy": "direct_json",
            "parsing_success": parsing_success,
            "parsing_error": parsing_error,
            "parsed_actions": parsed.get("actions", []) if parsed else [],
            "action_count": len(parsed.get("actions", [])) if parsed else 0,
            "validation_errors": [],
            "warnings": []
        }
    
    async def _debug_action_execution(self, actions: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Debug action execution phase (simulated)"""
        
        execution_results = []
        
        for action in actions:
            # Simulate action execution
            await asyncio.sleep(0.05)
            
            execution_results.append({
                "action_type": action.get("action_type"),
                "element_id": action.get("element_id"),
                "success": True,  # Simulated success
                "execution_time": 0.5,
                "result_message": "Action executed successfully (simulated)"
            })
        
        return {
            "phase": "action_execution",
            "actions_attempted": len(actions),
            "actions_successful": len(execution_results),
            "total_execution_time": sum(r["execution_time"] for r in execution_results),
            "execution_results": execution_results,
            "warnings": []
        }
    
    def _generate_debug_summary(self, session: Dict[str, Any]) -> Dict[str, Any]:
        """Generate comprehensive debug summary"""
        
        total_steps = len(session["steps"])
        successful_steps = sum(
            1 for step in session["steps"] 
            if not step.get("critical_error")
        )
        
        return {
            "total_steps_attempted": total_steps,
            "successful_steps": successful_steps,
            "success_rate": successful_steps / total_steps if total_steps > 0 else 0,
            "common_issues": self._identify_common_issues(session["steps"]),
            "performance_metrics": self._calculate_performance_metrics(session["steps"]),
            "recommendations": self._generate_recommendations(session["steps"])
        }
    
    def _identify_common_issues(self, steps: List[Dict[str, Any]]) -> List[str]:
        """Identify common issues across steps"""
        issues = []
        
        # Check for parsing failures
        parsing_failures = sum(
            1 for step in steps
            if not step.get("phases", {}).get("response_parsing", {}).get("parsing_success", True)
        )
        if parsing_failures > 0:
            issues.append(f"Response parsing failed in {parsing_failures} steps")
        
        # Check for slow LLM responses
        slow_responses = sum(
            1 for step in steps
            if step.get("phases", {}).get("llm_processing", {}).get("processing_time", 0) > 5.0
        )
        if slow_responses > 0:
            issues.append(f"Slow LLM responses in {slow_responses} steps")
        
        return issues
    
    async def _save_debug_session(self, session: Dict[str, Any]):
        """Save debug session to file"""
        
        debug_file = self.debug_dir / f"debug_session_{self.session_id}.json"
        
        with open(debug_file, 'w') as f:
            json.dump(session, f, indent=2, default=str)
        
        LOG.info(f"Debug session saved to {debug_file}")

# Usage example
debugger = LLMDebugger()
debug_result = await debugger.debug_complete_flow(
    task_id="task_123",
    user_goal="Login to user account",
    initial_url="https://example.com/login"
)
```

---

## 📋 Common Issues & Solutions

### **10. Troubleshooting Guide**

| Issue | Symptoms | Root Cause | Solution |
|-------|----------|------------|----------|
| **Slow LLM Responses** | >10s per decision | Large prompts, slow provider | Optimize prompts, use faster models |
| **Parsing Failures** | Invalid JSON responses | LLM output formatting issues | Implement robust parsing strategies |
| **High Error Rates** | >10% action failures | Element detection issues | Improve element selection logic |
| **Memory Leaks** | Increasing memory usage | Unclosed browser sessions | Implement proper cleanup |
| **Rate Limiting** | 429 HTTP errors | Exceeding API limits | Implement rate limiting, use multiple providers |

### **Quick Fixes**

```python
# Quick diagnostic script
async def diagnose_llm_integration():
    """Quick diagnostic for LLM integration issues"""
    
    print("🔍 Diagnosing LLM Integration...")
    
    # 1. Test API connectivity
    try:
        handler = LLMAPIHandlerFactory.get_llm_api_handler("primary-llm")
        test_response = await handler(
            prompt="Test prompt",
            prompt_name="diagnostic-test"
        )
        print("✅ LLM API connectivity: OK")
    except Exception as e:
        print(f"❌ LLM API connectivity: FAILED - {e}")
    
    # 2. Test prompt templates
    try:
        from skyvern.utils.prompt_engine import PromptEngine
        engine = PromptEngine("gpt-4o")
        test_prompt = engine.load_prompt("action-extraction", user_goal="test")
        print("✅ Prompt templates: OK")
    except Exception as e:
        print(f"❌ Prompt templates: FAILED - {e}")
    
    # 3. Test response parsing
    try:
        test_response = '{"actions": [{"action_type": "CLICK", "element_id": "test"}]}'
        parser = CustomResponseParser()
        actions = await parser.parse_response(test_response, {})
        print("✅ Response parsing: OK")
    except Exception as e:
        print(f"❌ Response parsing: FAILED - {e}")
    
    print("🎯 Diagnosis complete!")

# Run diagnostic
await diagnose_llm_integration()
```

---

## 🎓 Key Takeaways

1. **Robust Configuration** is essential for production deployments
2. **Comprehensive Monitoring** enables proactive issue resolution
3. **Effective Debugging Tools** accelerate development and troubleshooting
4. **Best Practices** ensure scalable and maintainable AI integration
5. **Error Handling** strategies improve system reliability

---

## 🚀 Next Steps

1. **Implement monitoring** in your Skyvern deployment
2. **Set up proper logging** for debugging and analysis
3. **Configure multiple LLM providers** for redundancy
4. **Test error scenarios** to validate recovery mechanisms
5. **Monitor performance metrics** and optimize accordingly

---

**Previous:** [AI Decision Making Flow ←](./05-ai-decision-flow.md)  
**Back to Main:** [Main Presentation ←](./skyvern_ai_llm_presentation_main.md)