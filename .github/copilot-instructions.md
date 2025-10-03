# DeerFlow AI Agent Instructions

## System Architecture

DeerFlow is a **LangGraph-based multi-agent research framework** that orchestrates specialized agents in a directed workflow graph. The system follows a **node-based state machine pattern** where each node represents a specialized agent with specific capabilities.

### Core Workflow Pattern
- **Graph Builder**: `src/graph/builder.py` defines the main workflow using `StateGraph(State)` 
- **State Management**: `src/graph/types.py` extends `MessagesState` with research-specific fields
- **Node Functions**: `src/graph/nodes.py` contains agent implementations that return `Command[Literal[...]]` objects
- **Conditional Routing**: Use `builder.add_conditional_edges()` for dynamic workflow branching

Key nodes: `coordinator → background_investigator → planner → research_team → reporter`

## Configuration System

The system uses a **dual-configuration approach**:

1. **YAML Config** (`conf.yaml`): LLM providers, models, reasoning models, search engines
2. **Environment Variables** (`.env`): API keys, tool toggles, provider selection

### Critical Config Patterns
- **Agent-LLM Mapping**: `src/config/agents.py` - `AGENT_LLM_MAP` assigns specific LLM types to agents
- **Configuration Class**: `src/config/configuration.py` - Use `Configuration.from_runnable_config(config)` in nodes
- **Tool Selection**: Environment variables like `SEARCH_API`, `RAG_PROVIDER` control tool availability

## Agent Development Patterns

### Creating New Agents
```python
# Use the factory function from src/agents/agents.py
agent = create_agent(
    agent_name="my_agent",
    agent_type="researcher",  # Must exist in AGENT_LLM_MAP
    tools=[tool1, tool2],
    prompt_template="my_prompt"  # References src/prompts/my_prompt.md
)
```

### Node Function Structure
```python
async def my_node(state: State, config: RunnableConfig) -> Command[Literal["next_node"]]:
    configurable = Configuration.from_runnable_config(config)
    # Agent logic here
    return Command(
        update={"state_field": new_value},
        goto="next_node"
    )
```

## Tool Integration

### Core Tools Pattern
- **Dynamic Tool Loading**: Tools are conditionally added based on config/environment
- **Logged Tools**: Use `create_logged_tool()` decorator from `src/tools/decorators.py` for I/O logging
- **Tool Selection**: `get_web_search_tool()`, `get_retriever_tool()` return configured instances

### Standard Tool Set
- `web_search`: Configurable search (Tavily/DuckDuckGo/Brave/Arxiv/Wikipedia)
- `crawl_tool`: URL content extraction with readability parsing  
- `python_repl_tool`: Code execution (disabled by default, enable with env var)
- `local_search_tool`: RAG retrieval from knowledge bases

## Prompt System

### Jinja2 Template Pattern
- **Location**: `src/prompts/*.md` files contain Jinja2 templates
- **Variables**: Use `{{ CURRENT_TIME }}`, `{{ locale }}`, `{{ research_topic }}` etc.
- **Application**: `apply_prompt_template(prompt_name, state, configurable)` returns message list
- **Agent-Specific**: Each agent type has dedicated prompt files (planner.md, researcher.md, etc.)

## Multi-Modal Workflows

The codebase includes **specialized sub-graphs**:
- **Podcast**: `src/podcast/graph/` - Script generation → TTS → Audio mixing
- **PPT**: `src/ppt/graph/` - Content composition → Slide generation  
- **Prose**: `src/prose/graph/` - Text enhancement with conditional routing
- **Prompt Enhancement**: `src/prompt_enhancer/graph/` - Single-node workflow

### Sub-Graph Pattern
Each has its own `State` class extending `MessagesState` and dedicated `builder.py` with workflow-specific logic.

## Development Workflows

### Running the System
- **Console**: `uv run main.py` (uses built-in questions)
- **Web UI**: `./bootstrap.sh -d` (starts both backend/frontend)
- **Server Only**: `uv run server.py --host 127.0.0.1`

### Debugging with LangGraph Studio
```bash
# Install and run LangGraph Studio for visual workflow debugging
uvx --from "langgraph-cli[inmem]" --python 3.12 langgraph dev --allow-blocking
```
- Uses `langgraph.json` config file for graph structure
- Visualize execution flow, inspect state at each step
- Interactive human feedback during planning phase

### Testing Patterns
- **Unit Tests**: `tests/unit/` - Mock external dependencies
- **Integration Tests**: `tests/integration/` - End-to-end workflows  
- **State Testing**: Use stub classes to avoid import cascades

## Key Conventions

### Error Handling
- Nodes should return meaningful error states rather than throwing exceptions
- Use `logger.error()` for debugging with structured messages
- Tool failures should be gracefully handled in agent logic

### State Updates
- Always use `Command(update={...}, goto="next_node")` for state transitions
- Preserve message history in `state.get("messages", [])`
- Use type hints: `Command[Literal["node1", "node2", "__end__"]]`

### Memory Management
- **With Memory**: Use `build_graph_with_memory()` for conversation persistence
- **Stateless**: Use `build_graph()` for single-shot execution
- Memory uses `MemorySaver()` (TODO: SQLite/PostgreSQL compatibility)

## Extension Points

### Adding New Search Engines
1. Add enum to `src/config/tools.py` `SearchEngine`
2. Implement in `src/tools/search.py` `get_web_search_tool()`
3. Use `create_logged_tool()` wrapper for consistency

### Adding New Agents
1. Define LLM mapping in `src/config/agents.py`
2. Create prompt template in `src/prompts/`
3. Add node function to `src/graph/nodes.py`
4. Update graph builder with new edges

### RAG Integration
- Implement `src/rag/*.py` provider classes
- Add to `RAGProvider` enum in config
- Tools automatically detect and use available providers