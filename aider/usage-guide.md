# Aider

## Env. setup (before running)
```bash
# Deepseek/Ollama specific
export OLLAMA_API_BASE=http://127.0.0.1:11434

# Optional
export AIDER_ATTRIBUTE_CO_AUTHORED_BY=false
export AIDER_ATTRIBUTE_AUTHOR=false
export AIDER_ATTRIBUTE_COMMITTER=false
```

## Running
You can run Aider by executing the following command:
```bash
aider --model ollama/deepseek-r1:70b
```
inside the GitHub repo you want to use Aider with.


## Common commands
### File & Context Management
* **`/add <file>`**: Adds a file to the chat so the AI can read and edit it. You can use wildcards (e.g., `/add src/*.py`).
* **`/drop <file>`**: Removes a file from the AI's context. Do this when you are done working on a file to save your RAM and keep the AI focused.
* **`/read <file>`**: Adds a file as "read-only." The AI will use it for context (like reading API documentation or a configuration file) but will not attempt to edit it.
* **`/ls`**: Lists all files currently in the chat so you can verify what the AI is currently looking at.
* **`/clear`**: Wipes the entire chat history. This is essential when switching tasks, as it clears out the AI's short-term memory, preventing it from getting confused by old prompts and freeing up your context window.

### Coding & Architecture Modes
* **`/ask <question>`**: Asks the AI a question without allowing it to make any code edits. Perfect for asking "Can you explain how this function works?" without risking accidental changes.
* **`/architect`**: Enters "Architect Mode." You propose a complex change, the AI will "think" and write a detailed architectural plan, and then ask for your approval. Once you approve, it switches to a coding mode to execute the plan.
* **`/chat-mode <mode>`**: Manually switches between modes (`code`, `ask`, `architect`, `help`).
* **`/web <url>`**: Scrapes the text from a webpage and adds it to the chat context. Incredibly useful for instantly feeding the AI the newest documentation for a framework.

### Git & Safety Controls
* **`/undo`**: The most important command. It instantly reverts the last AI edit and removes the Git commit, returning your codebase exactly to how it was before the previous prompt.
* **`/diff`**: Shows the standard Git diff of the uncommitted changes or the last AI edit, so you can review exactly what was altered line-by-line.
* **`/commit`**: Manually commits any pending changes in your workspace with an AI-generated commit message.
* **`/git <command>`**: Runs a standard Git command directly through Aider (e.g., `/git status` or `/git log`).

### System Execution & Self-Healing
* **`!<command>`** or **`/run <command>`**: Runs a shell command directly in your Ubuntu terminal. For example, `!python3 main.py` or `!ls -la`.
* **`/test <command>`**: Runs your testing suite (e.g., `/test pytest`). If the tests fail, Aider automatically feeds the error logs back to the AI, and the AI will attempt to write a fix and run the tests again, looping until the tests pass.
* **`/lint <command>`**: Similar to `/test`, this runs your linter (e.g., `/lint flake8`). The AI will automatically read the linting errors and fix formatting and syntax issues in your files.

### Help & Troubleshooting
* **`/help <question>`**: Asks the AI a question specifically about how to use Aider, its commands, or its configuration. It references Aider's internal documentation.
* **`/tokens`**: Displays exactly how many tokens (words) are currently being used by your files and chat history. This is vital for monitoring your 64GB RAM/GPU memory budget.
