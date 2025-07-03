# 🧠 Skyvern AI & LLM Integration
## Phase 6: Deep Dive Technical Presentation

---

## 📋 Table of Contents

1. **[Introduction & Overview](./01-introduction.md)**
   - What is Skyvern's AI Integration?
   - Key Components Overview
   - Architecture High-level View

2. **[LLM API Integration](./02-llm-api-integration.md)**
   - LLM Provider Management
   - API Handler Factory
   - Configuration Registry
   - Multi-provider Support

3. **[Prompt Engineering System](./03-prompt-engineering.md)**
   - Prompt Template Management
   - Dynamic Prompt Generation
   - Jinja2 Template Engine
   - Context-aware Prompting

4. **[AI Response Processing](./04-ai-response-processing.md)**
   - Response Parsing Pipeline
   - Action Extraction
   - Validation & Error Handling
   - UI-TARS Integration

5. **[AI Decision Making Flow](./05-ai-decision-flow.md)**
   - Complete AI Workflow
   - Decision Points
   - Feedback Loops
   - Performance Optimization

6. **[Implementation Examples](./06-implementation-examples.md)**
   - Code Examples
   - Configuration Samples
   - Best Practices
   - Troubleshooting

---

## 🎯 Learning Objectives

By the end of this presentation, you will understand:

✅ **How AI models are integrated** into Skyvern's architecture  
✅ **Prompt engineering patterns** and template management  
✅ **Response parsing and validation** mechanisms  
✅ **AI decision-making process** and optimization strategies  
✅ **Best practices** for LLM integration in automation systems  

---

## 🔧 Prerequisites

- Basic understanding of Large Language Models (LLMs)
- Familiarity with Python async/await patterns
- Knowledge of web automation concepts
- Understanding of JSON data structures

---

## 📚 Key Files Referenced

```
skyvern/
├── forge/sdk/api/llm/          # 🔥 CRITICAL - LLM Integration Core
│   ├── api_handler_factory.py   # LLM Handler Management
│   ├── config_registry.py       # Configuration Management
│   ├── models.py                # Data Models
│   └── utils.py                 # Utility Functions
├── utils/prompt_engine.py       # 🔥 CRITICAL - Prompt Management
├── forge/prompts/              # 🔥 IMPORTANT - Prompt Templates
└── webeye/actions/parse_actions.py # 🔥 CRITICAL - Response Parsing
```

---

**Next:** [Introduction & Overview →](./01-introduction.md)