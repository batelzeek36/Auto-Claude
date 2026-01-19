# Bug: Task Continues Executing After Being Moved to Planning

**Date Identified**: 2026-01-18  
**Status**: Identified, fix pending  
**Severity**: High - Core functionality broken

## Symptom

When a user drags a running task from "In Progress" back to "Planning" column, the task's UI status changes to Planning but the underlying process continues executing.

## Root Cause

**Task ID mismatch between spec creation and task execution tracking.**

### The Problem

In `apps/frontend/src/main/ipc-handlers/task/execution-handlers.ts`:

**Spec creation** (line 213) registers process under `task.specId`:
```typescript
agentManager.startSpecCreation(task.specId, project.path, taskDescription, specDir, task.metadata, baseBranch);
```

**Task execution** (lines 228, 244) registers process under `task.id`:
```typescript
agentManager.startTaskExecution(taskId, project.path, task.specId, { ... });
```

**Auto-stop logic** (line 687) only checks `task.id`:
```typescript
if (status !== 'in_progress' && agentManager.isRunning(taskId)) {
  agentManager.killTask(taskId);
}
```

### What Happens

1. User starts a new task → spec creation runs, process tracked under `task.specId`
2. User drags task to Planning column → TASK_UPDATE_STATUS handler called
3. `agentManager.isRunning(taskId)` checks for `task.id` 
4. Returns `false` because process is registered under `task.specId`
5. Kill logic is **skipped**
6. Process continues running

### Secondary Issue

The auto-stop path doesn't call `fileWatcher.unwatch(taskId)` even when it does kill a process, unlike the TASK_STOP handler which properly unwatches.

## Affected Files

- `apps/frontend/src/main/ipc-handlers/task/execution-handlers.ts`

## Recommended Fix

### Option A: Consistent Task ID Usage (Recommended)

Change `startSpecCreation` calls to use `taskId` instead of `task.specId`:

**Line 213** - Change from:
```typescript
agentManager.startSpecCreation(task.specId, project.path, taskDescription, specDir, task.metadata, baseBranch);
```
To:
```typescript
agentManager.startSpecCreation(taskId, project.path, taskDescription, specDir, task.metadata, baseBranch);
```

**Line 756** - Change from:
```typescript
agentManager.startSpecCreation(task.specId, project.path, taskDescription, specDir, task.metadata, baseBranchForUpdate);
```
To:
```typescript
agentManager.startSpecCreation(taskId, project.path, taskDescription, specDir, task.metadata, baseBranchForUpdate);
```

**Line 1124** - Change from:
```typescript
agentManager.startSpecCreation(task.specId, project.path, taskDescription, specDirForWatcher, task.metadata, baseBranchForRecovery);
```
To:
```typescript
agentManager.startSpecCreation(taskId, project.path, taskDescription, specDirForWatcher, task.metadata, baseBranchForRecovery);
```

### Also Required: Add fileWatcher.unwatch()

After line 689, add:
```typescript
if (status !== 'in_progress' && agentManager.isRunning(taskId)) {
  console.warn('[TASK_UPDATE_STATUS] Stopping task due to status change away from in_progress:', taskId);
  agentManager.killTask(taskId);
  fileWatcher.unwatch(taskId);  // ADD THIS LINE
}
```

## Testing

After fix:
1. Create a new task
2. Start the task (it should begin spec creation)
3. While spec creation is running, drag the task back to Planning column
4. Verify the Python process is terminated
5. Verify no more logs appear for that task
