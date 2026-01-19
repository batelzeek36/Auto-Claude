# Bug Report: Silent IPC failure after update - requires manual rebuild to fix

## Summary

After pulling the latest updates, the Kanban board drag-to-start functionality stops working silently. Tasks can be dragged to "In Progress" visually, but no agent execution occurs. The UI shows 0% progress indefinitely with "No logs yet" message. No error is displayed to the user.

## Environment

- **OS:** macOS (Darwin 25.2.0)
- **Auto-Claude Version:** 2.7.5
- **Node.js Version:** (run `node -v` to fill in)
- **Recent commits pulled:** Biome migration (#1289), Token Decryption fix (#1283), and others

## Steps to Reproduce

1. Pull latest changes from the repository (`git pull`)
2. Start the app with `npm run dev` or `npm start`
3. Open a project with existing tasks
4. Drag a task from Backlog/Planning to "In Progress" column
5. Observe: Task card moves visually, but nothing happens - no logs, no progress, no agent starts

## Expected Behavior

- Task should start executing
- Logs should appear in the task detail view
- Progress should update from 0%
- Console should show `[TASK_UPDATE_STATUS] Auto-starting task:` log

## Actual Behavior

- Task appears stuck at 0% progress
- "No logs yet - Logs will appear here when the task runs" message persists
- No console logs related to task execution appear
- DevTools console shows normal app startup but NO IPC-related logs when dragging tasks
- Debug info panel shows old EPIPE errors (not directly related)

## Root Cause Analysis

The IPC (Inter-Process Communication) bridge between the renderer process and main process appears to silently fail. When `persistTaskStatus()` is called from the Kanban board, the `ipcRenderer.invoke()` call either:
- Never reaches the main process handler
- Returns without triggering the auto-start logic

This is likely caused by a mismatch between the compiled JavaScript bundles and the updated source code after pulling changes.

## Workaround / Fix

Running a full rebuild resolves the issue:

```bash
cd /path/to/Auto-Claude
npm run build
# Then restart the app
npm run dev
```

After rebuild, tasks start executing normally.

## Suggested Improvements

1. **Auto-rebuild detection:** Check if source files are newer than compiled output on startup, and trigger rebuild if needed

2. **IPC failure handling:** Add error handling/timeout for IPC calls so users see an error message instead of silent failure:
   ```typescript
   // In task-store.ts persistTaskStatus()
   const result = await Promise.race([
     window.electronAPI.updateTaskStatus(taskId, status, options),
     new Promise((_, reject) =>
       setTimeout(() => reject(new Error('IPC timeout')), 5000)
     )
   ]);
   ```

3. **Startup validation:** Add a simple IPC ping/pong check on app startup to verify the bridge is working

4. **Documentation:** Add a note in README or CONTRIBUTING about running `npm run build` after pulling updates

## Screenshots

### Before fix - Task stuck at 0%
Task shows "In Progress" status but with:
- 0/11 subtasks
- 0% progress
- "No logs yet" message

### After fix - Task completes normally
Task progresses through all phases and moves to "AI Review" at 100%

## Additional Context

The issue appeared after pulling updates that included the Biome migration (ESLint to Biome, #1289). The migration touched many files which may have contributed to the build cache invalidation issue.

The most confusing aspect was the **complete silence** - no errors, no warnings, no indication that something was broken. The app appeared to work normally except tasks wouldn't start.
