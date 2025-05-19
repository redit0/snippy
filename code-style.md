# Project Code Style Guide

This style guide is synthesized from the project's code snippets and best practices. It aims to ensure consistency, readability, and maintainability across all code contributions.

## 1. Naming Conventions
- **Modules & Files:** Use `snake_case` for Python files and modules.
- **Classes:** Use `CamelCase` for class names.
- **Functions & Methods:** Use `snake_case` for function and method names.
- **Constants:** Use `UPPER_SNAKE_CASE` for constants.
- **Variables:** Use descriptive `snake_case` names for variables.
- **Environment Variables:** Use `UPPER_SNAKE_CASE` and access via `os.environ[...]`.

## 2. Code Organization
- Group related functions and classes into modules.
- Place configuration and environment variable access at the top of the file.
- Use `__init__.py` to define package structure if needed.
- Keep function length manageable; refactor long functions into smaller helpers.

## 3. Documentation Standards
- Every public function, class, and module should have a docstring.
- Use [Google-style docstrings](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings):
  ```python
  def example(arg1: int) -> str:
      """One-line summary.
      
      Args:
          arg1: Description of argument.
      Returns:
          Description of return value.
      """
  ```
- Document all arguments, return values, and exceptions.

## 4. Error Handling
- Use `try`/`except` blocks to handle expected errors.
- Log errors using the `logging` module at the appropriate level (`info`, `warning`, `error`).
- Never use `print` for error reporting; always use `logging`.
- Raise exceptions with clear, actionable messages.
- Do not ignore caught exceptions; log or handle them appropriately.

## 5. Logging Practices
- Use the `logging` module, not `print`.
- Configure loggers at the module level:
  ```python
  import logging
  logger = logging.getLogger(__name__)
  ```
- Set Azure SDK loggers to `WARNING` to reduce noise:
  ```python
  logging.getLogger("azure").setLevel(logging.WARNING)
  logging.getLogger("azure.core").setLevel(logging.WARNING)
  logging.getLogger("azure.ai.projects").setLevel(logging.WARNING)
  ```
- Log key events, errors, and important state changes.

## 6. Async & Await
- Use `async def` for functions that perform I/O or network operations.
- Use `await` only with async functions.
- Do not `await` synchronous methods (e.g., `req.get_json()`).
- Use `unittest.mock.AsyncMock` for patching async functions in tests.

## 7. Azure SDK & Environment
- Prefer async Azure SDK clients (e.g., `BlobServiceClient.from_connection_string`).
- Access secrets and connection strings via environment variables, never hard-coded.
- Use `DefaultAzureCredential` for authentication in Azure environments.

## 8. Testing
- Use `pytest` and `pytest-asyncio` for tests.
- Mark async tests with `@pytest.mark.asyncio`.
- Patch Azure SDK calls with `AsyncMock` or `MagicMock`.
- Aim for 100% branch coverage, especially for orchestrator fan-out/fan-in paths.
- Keep test payloads small and inline.

## 9. Miscellaneous
- Use Python 3.11+ features (e.g., `match` statements) where appropriate.
- Avoid global variables for managing state.
- Do not mix business logic with presentation logic.
- Use type hints throughout the codebase.

## 10. Example: AI Agent Service Usage
```python
import os
import logging
from azure.ai.projects.aio import AIProjectClient
from azure.ai.projects.models import AsyncFunctionTool
from azure.identity.aio import DefaultAzureCredential
from agents.tools import vector_search

logger = logging.getLogger(__name__)
logging.getLogger("azure").setLevel(logging.WARNING)
logging.getLogger("azure.core").setLevel(logging.WARNING)
logging.getLogger("azure.ai.projects").setLevel(logging.WARNING)

async def generate_code_style(chat_history: str = "", user_query: str = "") -> str:
    """Generates a code style guide using an AI agent."""
    try:
        logger.info("Starting code style generation with:")
        logger.info("Chat history length: %d characters", len(chat_history))
        if chat_history:
            logger.info("Chat history preview: %s", chat_history[:200] + "..." if len(chat_history) > 200 else chat_history)
        logger.info("User query: %s", user_query)
        async with DefaultAzureCredential() as credential:
            async with AIProjectClient.from_connection_string(
                credential=credential,
                conn_str=os.environ["PROJECT_CONNECTION_STRING"]
            ) as project_client:
                functions = AsyncFunctionTool(functions=[vector_search.vector_search])
                agent = await project_client.agents.create_agent(
                    name="CodeStyleSynthesizer",
                    description="An agent that produces code style guides",
                    instructions="...",
                    tools=functions.definitions,
                    model=os.environ["AGENTS_MODEL_DEPLOYMENT_NAME"]
                )
                thread = await project_client.agents.create_thread()
                if chat_history:
                    await project_client.agents.create_message(
                        thread_id=thread.id,
                        role="user",
                        content=chat_history
                    )
                final_query = user_query if user_query else "Generate a code style guide."
                await project_client.agents.create_message(
                    thread_id=thread.id,
                    role="user",
                    content=final_query
                )
                run = await project_client.agents.create_run(
                    thread_id=thread.id,
                    agent_id=agent.id
                )
                tool_call_count = 0
                while True:
                    run = await project_client.agents.get_run(thread_id=thread.id, run_id=run.id)
                    if run.status == "completed":
                        break
                    elif run.status == "failed":
                        raise Exception("Agent run failed")
                    elif run.status == "requires_action":
                        tool_calls = run.required_action.submit_tool_outputs.tool_calls
                        tool_outputs = []
                        for tool_call in tool_calls:
                            output = await functions.execute(tool_call)
                            tool_outputs.append({
                                "tool_call_id": tool_call.id,
                                "output": output
                            })
                            tool_call_count += 1
                        await project_client.agents.submit_tool_outputs_to_run(
                            thread_id=thread.id,
                            run_id=run.id,
                            tool_outputs=tool_outputs
                        )
                messages = await project_client.agents.list_messages(thread_id=thread.id)
                response = str(messages.data[0].content[0].text.value)
                return response
    except Exception as e:
        logger.error("Code style generation failed with error: %s", str(e), exc_info=True)
        raise
```

---

Adhering to this guide will help maintain a high-quality, consistent codebase that is easy to review, test, and extend.
