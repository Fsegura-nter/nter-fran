# nter-usage:usage

Launch the claude-usage dashboard on port 9090. Auto-closes after 10 minutes.

---

## What this does

1. Stops any previous dashboard instance (if running)
2. Starts the dashboard on port 9090 in the background
3. Schedules auto-shutdown after 10 minutes
4. Reports the URL

---

## Steps

### 1. Start (or restart) the dashboard

Run the following as a single background command:

```bash
PID_FILE="$TEMP/claude-usage.pid"

# Kill previous instance if any
if [ -f "$PID_FILE" ]; then
  OLD_PID=$(cat "$PID_FILE")
  kill "$OLD_PID" 2>/dev/null
  rm -f "$PID_FILE"
fi

# Start dashboard on port 9090
PORT=9090 python "C:/Users/EM2025007530/Desktop/Proyectos/Misc/claude-usage/cli.py" dashboard &
USAGE_PID=$!
echo $USAGE_PID > "$PID_FILE"

# Auto-kill after 10 minutes
(sleep 600 && kill $(cat "$TEMP/claude-usage.pid" 2>/dev/null) 2>/dev/null && rm -f "$TEMP/claude-usage.pid") &
```

### 2. Report to user

After launching, output exactly:

```
Dashboard running at http://localhost:9090
Auto-closes in 10 minutes. Run /nter-usage:usage again to reset the timer.
```

The browser opens automatically.
