---
name: non-blocking-execution
description: This skill enables the agent to correctly handle long-running processes (daemons, servers, watchers) that would otherwise block the CLI. It provides standard patterns for starting background jobs, capturing logs, saving PIDs, and performing graceful shutdowns.
---

# NON-BLOCKING EXECUTION PROTOCOL

Use this protocol when you need to run commands that do not exit immediately (e.g., `npm start`, `python server.py`, `streamlit run`, `docker run`).

## NAMING CONVENTION

**APP_ID must use `snake_case`** (e.g., `backend_api`, `streamlit_app`, `node_server`). This identifier is used for PID and log files.

## 1. PORT CONFLICT CHECK (Pre-flight)

Before starting a server, verify the target port is available to avoid "Address already in use" errors.

```bash
# Check if port 8501 is free
lsof -i :8501 > /dev/null && echo "PORT 8501 IN USE" || echo "PORT 8501 AVAILABLE"

# Or kill process using the port
lsof -ti :8501 | xargs kill -9 2>/dev/null || echo "Port already free"
```

## 2. START PATTERN (Background Execution)

Never run a blocking command directly. Always redirect stdout/stderr to a log file, run in the background, and save the Process ID (PID).

**Template:**
```bash
[COMMAND] > /tmp/[APP_ID].log 2>&1 & echo $! > /tmp/[APP_ID].pid
```

- `[COMMAND]`: The actual command (e.g., `python3 app.py`)
- `[APP_ID]`: Unique `snake_case` identifier (e.g., `backend_api`, `streamlit_app`)

**Example:**
```bash
streamlit run app.py > /tmp/streamlit_app.log 2>&1 & echo $! > /tmp/streamlit_app.pid
```

## 3. VERIFY PATTERN (Health Check)

Immediately after starting, verify the process is still running and check the initial logs. **Adjust sleep duration based on startup time** (Streamlit: 5s, Spring Boot: 10-15s, Node: 3s).

**Template:**
```bash
sleep [SECONDS] && ps -p $(cat /tmp/[APP_ID].pid) > /dev/null && echo "STATUS: RUNNING" || echo "STATUS: FAILED"
```

**Example:**
```bash
# Streamlit (typical startup ~5s)
sleep 5 && ps -p $(cat /tmp/streamlit_app.pid) > /dev/null && echo "STATUS: RUNNING" || echo "STATUS: FAILED"

# Spring Boot (typical startup ~10-15s)
sleep 12 && ps -p $(cat /tmp/backend_api.pid) > /dev/null && echo "STATUS: RUNNING" || echo "STATUS: FAILED"
```

## 4. DEBUG PATTERN (Log Retrieval)

To check for startup errors, tokens, or local URLs printed by the server.

**View full log:**
```bash
cat /tmp/[APP_ID].log
```

**Tail recent output (last 50 lines):**
```bash
tail -50 /tmp/[APP_ID].log
```

**Follow live logs:**
```bash
tail -f /tmp/[APP_ID].log
```

**Example:**
```bash
tail -50 /tmp/streamlit_app.log
```

## 5. LOG ROTATION (Cleanup)

Prevent `/tmp` pollution by cleaning old logs when starting fresh instances.

**Template:**
```bash
# Clear old log before starting
> /tmp/[APP_ID].log

# Or remove entirely
rm -f /tmp/[APP_ID].log
```

**Integrated example:**
```bash
# Clean old log, then start fresh
> /tmp/streamlit_app.log
streamlit run app.py > /tmp/streamlit_app.log 2>&1 & echo $! > /tmp/streamlit_app.pid
```

## 6. STOP PATTERN (Graceful Cleanup)

Before starting a new instance of the same application, or when finished testing, always kill the existing process using the stored PID.

**Template:**
```bash
if [ -f /tmp/[APP_ID].pid ]; then 
  kill $(cat /tmp/[APP_ID].pid) && rm /tmp/[APP_ID].pid && echo "Process terminated"; 
else 
  echo "No active process found"; 
fi
```

**Example:**
```bash
if [ -f /tmp/streamlit_app.pid ]; then 
  kill $(cat /tmp/streamlit_app.pid) && rm /tmp/streamlit_app.pid && echo "Process terminated"; 
else 
  echo "No active process found"; 
fi
```

## COMPLETE WORKFLOW EXAMPLE

**Scenario: Starting a Streamlit app on port 8501**

```bash
# 1. Check port availability
lsof -ti :8501 | xargs kill -9 2>/dev/null || echo "Port 8501 ready"

# 2. Cleanup old instance
if [ -f /tmp/streamlit_app.pid ]; then 
  kill $(cat /tmp/streamlit_app.pid) 2>/dev/null
  rm /tmp/streamlit_app.pid
fi

# 3. Clear old logs
> /tmp/streamlit_app.log

# 4. Start new instance
streamlit run app.py > /tmp/streamlit_app.log 2>&1 & echo $! > /tmp/streamlit_app.pid

# 5. Verify startup (wait 5s for Streamlit)
sleep 5 && ps -p $(cat /tmp/streamlit_app.pid) > /dev/null && echo "STATUS: RUNNING" || echo "STATUS: FAILED"

# 6. Check logs for URL/errors
tail -50 /tmp/streamlit_app.log
```

## COMMON APP_ID EXAMPLES

| Application | Recommended APP_ID | Typical Wait Time |
|-------------|-------------------|-------------------|
| Streamlit | `streamlit_app` | 5s |
| FastAPI/Uvicorn | `fastapi_server` | 3s |
| Spring Boot | `backend_api` | 10-15s |
| Node.js/Express | `node_server` | 3s |
| React dev server | `react_dev` | 5s |
| Django | `django_server` | 5s |
| Docker container | `docker_[service]` | varies |

## TROUBLESHOOTING

**Process starts then immediately dies:**
```bash
# Check exit logs
cat /tmp/[APP_ID].log
```

**Port already in use:**
```bash
# Find and kill process on port
lsof -ti :[PORT] | xargs kill -9
```

**PID file exists but process is dead:**
```bash
# Clean stale PID
rm /tmp/[APP_ID].pid
```
