# Workplan 001: Fix Claude Launch Crash + Custom Path Support

## Problem

Crystal Electron app crashes silently when Claude CLI is not in PATH. The crash occurs during initialization in `main/src/index.ts` at line ~575 when `cliManagerFactory.createManager('claude', ...)` throws an error before the window is created.

## Solution

Minimal fix with 3 stages, ~7 lines changed total. Each stage leaves the app in a functional state.

---

## Stage 1: Prevent Crash with Try-Catch ✅

**Goal**: Prevent app crash by catching the error during initialization

**File**: `/Users/cazer/dev/crystal/main/src/index.ts`

**Location**: Line ~575

**Before:**
```typescript
// Create default CLI manager (Claude) with permission IPC path
defaultCliManager = await cliManagerFactory.createManager('claude', {
  sessionManager,
  logger,
  configManager,
  additionalOptions: { permissionIpcPath }
});
```

**After:**
```typescript
// Create default CLI manager (Claude) with permission IPC path
try {
  defaultCliManager = await cliManagerFactory.createManager('claude', {
    sessionManager,
    logger,
    configManager,
    additionalOptions: { permissionIpcPath }
  });
} catch (error) {
  logger?.warn(`Failed to initialize default Claude manager: ${error instanceof Error ? error.message : String(error)}`);
  logger?.warn('App will continue without pre-initialized Claude manager. Claude panels will initialize on first use.');
  defaultCliManager = null;
}
```

**Why it keeps app functional**:
- The app window will now launch successfully even if Claude CLI is missing
- `defaultCliManager` becomes `null` instead of crashing the entire app
- Sessions can still be created - they'll fail gracefully when trying to spawn Claude
- The error is logged for debugging without preventing app launch

---

## Stage 2: Read Custom Path from Config ✅

**Goal**: Make `testCliAvailability()` respect the `claudeExecutablePath` config setting

**File**: `/Users/cazer/dev/crystal/main/src/services/panels/cli/AbstractCliManager.ts`

**Location**: Line ~460

**Before:**
```typescript
// Perform fresh check
const availability = await this.testCliAvailability();
```

**After:**
```typescript
// Perform fresh check - pass custom path from config if available
const customPath = this.configManager?.getConfig()?.claudeExecutablePath;
const availability = await this.testCliAvailability(customPath);
```

**Why it keeps app functional**:
- Existing behavior is preserved (when `claudeExecutablePath` is undefined, it defaults to PATH search)
- When a custom path IS configured in `~/.crystal/config.json`, it will now be used
- No breaking changes - the parameter is already optional (`customPath?: string`)
- `ClaudeCodeManager.testCliAvailability()` already handles this parameter correctly
- The same fix applies to all CLI managers (Claude, Codex, future tools) via inheritance

---

## Stage 3: Make Manager Type Nullable ✅

**Goal**: Ensure the codebase handles `null` defaultCliManager gracefully

**File**: `/Users/cazer/dev/crystal/main/src/index.ts`

**Location**: Line ~46 (where defaultCliManager is declared)

**Before:**
```typescript
let defaultCliManager: AbstractCliManager;
```

**After:**
```typescript
let defaultCliManager: AbstractCliManager | null;
```

**Why it keeps app functional**:
- TypeScript will now enforce null checks wherever `defaultCliManager` is used
- This makes the codebase safer and more explicit about the manager being optional
- Existing code that uses `defaultCliManager` will need null checks, but the app won't silently fail

---

## Summary

**Total changes**: ~7 lines across 2 files

**Files Modified**:
- `/Users/cazer/dev/crystal/main/src/index.ts` (Stages 1 & 3)
- `/Users/cazer/dev/crystal/main/src/services/panels/cli/AbstractCliManager.ts` (Stage 2)

**Benefits**:
1. App launches even when Claude CLI is not installed
2. Respects custom `claudeExecutablePath` from `~/.crystal/config.json`
3. Merge-friendly minimal changes that won't conflict with main branch
4. TypeScript enforces null safety for optional manager

**Testing**:
- App should launch without Claude CLI in PATH
- App should respect custom path in config: `{"claudeExecutablePath": "/path/to/claude"}`
- Existing functionality with Claude in PATH should continue working
