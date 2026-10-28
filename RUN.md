# Running DocFusion AI on Windows

## First time
1. Install Python 3.11 or newer.
2. Install Ollama if you want Private AI.
3. Double-click `start.bat`.

The launcher creates the virtual environment, installs dependencies, creates `.env`, starts Ollama when available, starts FastAPI, and opens the browser.

For Private AI, install a local model once, for example:

```powershell
ollama pull gemma3:4b
```

For Public AI, put your Gemini API key in `.env` once:

```env
GEMINI_API_KEY=your_key_here
```

## Every later run
Just double-click `start.bat`.

## Stop
Close the launcher window, or double-click `stop.bat`.
