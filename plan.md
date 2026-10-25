# Android Bot Runner — Implementation Plan

**Project:** Local Sequential Android Automation  
**Target:** macOS · Apple Silicon M4 · Android Emulator  
**Initial scale:** 3 bots → 10 bots → 100 bots  
**Execution model:** Sequential, one task per bot  
**Status:** Implementation planning

## 1. Project objective

Build a local automation harness that manages 100 independent bot identities. Each bot executes exactly one defined interaction against an Android application, such as scrolling and clicking a target element, and produces a verifiable result.

The system must preserve each bot's app data, preferences, cache, and login session between runs. It must also support interruption, recovery, logging, and inspection of failed tasks.

### Goals

- Run 100 bot tasks sequentially on one MacBook Air M4.
- Isolate each bot's persistent Android application state.
- Reuse a common Android system image and automation code.
- Record every task's status and result in a local database.
- Avoid accidental duplicate execution after crashes.
- Verify task outcomes rather than assuming a successful click means success.
- Measure storage use and execution time before scaling up.

### Non-goals

- Running 100 emulators concurrently.
- Deploying a cloud service or distributed cluster.
- Automatically creating third-party accounts or bypassing application restrictions.
- Implementing a general-purpose AI agent.
- Guaranteeing distinct hardware-backed device identities through simple image cloning.

## 2. Proposed technology stack

| Component | Technology |
|---|---|
| Host operating system | macOS on Apple Silicon |
| Android runtime | Android Emulator |
| Device control | Android Debug Bridge (ADB) |
| UI automation | AndroidX UI Automator or app-specific test instrumentation |
| Orchestration | Python 3 |
| Persistent task registry | SQLite |
| Bot state | One writable emulator data image per bot |
| Logs and evidence | Local files and optional screenshots |
| Configuration | JSON |

Start with Python's standard library for orchestration and SQLite. Add AndroidX UI Automator where robust element selection and verification are needed.

## 3. Architecture

The system has four main components.

### 3.1 Bot registry

A JSON file defines the initial bot IDs and task configuration. SQLite stores runtime status, execution phase, timestamps, attempts, state-image path, and errors.

Each bot has a stable ID such as `bot-0001`.

### 3.2 Sequential orchestrator

The runner selects the next pending bot, starts its emulator using the corresponding writable data image, checks readiness, executes the task, verifies the result, records the outcome, and shuts down the emulator.

Only one emulator may run at a time.

### 3.3 Android automation layer

The automation layer exposes operations such as:

- `launch_app()`
- `wait_for_ready()`
- `scroll()`
- `click_target()`
- `verify_result()`
- `capture_screenshot()`

Prefer accessibility-based selectors or UI Automator over hardcoded screen coordinates when possible.

### 3.4 Persistence and recovery

SQLite is the authoritative task-status record. Each bot's writable data image is preserved independently. Screenshots and diagnostic logs are stored outside the database.

The runner must distinguish an ordinary failure from an uncertain outcome in which the action may have executed before the process crashed.

## 4. State isolation strategy

Each bot receives a distinct writable Android data image. The common Android system image, emulator binary, APK source, and automation code are shared.

Example:

    android-bot-runner/
    ├── runner.py
    ├── bots.json
    ├── config.json
    ├── app.apk
    ├── results.sqlite
    ├── state/
    │   ├── bot-0001/userdata.img
    │   ├── bot-0002/userdata.img
    │   └── ...
    ├── logs/
    └── screenshots/

### Isolation requirements

1. Never attach the same writable image to two emulator instances simultaneously.
2. Never overwrite an existing image when initializing a bot.
3. Install and provision the application independently in each image.
4. Use approved test accounts and test fixtures for login provisioning.
5. Verify that preferences, app data, cache, and session state remain isolated after shutdown and restart.
6. Treat cloned images as potentially sharing identifiers and credentials; do not assume that cloned images represent distinct devices.
7. Keep secrets out of source control and logs.

For the initial implementation, use fresh images. Evaluate baseline cloning only after confirming that it preserves the required isolation properties.

## 5. Task lifecycle and database model

Recommended status values:

- `pending`: Not yet executed.
- `running`: A task has been claimed.
- `succeeded`: The expected outcome was verified.
- `failed`: A failure occurred before an uncertain side effect.
- `uncertain`: The action may have executed, but its outcome cannot be established.
- `needs_review`: Manual inspection or recovery is required.

Recommended fields:

- `id`
- `status`
- `state_path`
- `attempts`
- `phase`
- `started_at`
- `finished_at`
- `error`
- `result_summary`

Persist every transition immediately.

On restart, any task left in `running` becomes `uncertain`. Never automatically repeat a potentially non-idempotent action without first inspecting its state.

## 6. Implementation phases

### Phase 1 — Three-bot proof of concept

**Objective:** Prove that independent persistent state works.

Tasks:

1. Install Android Studio, Android Emulator, and Platform-Tools.
2. Create one compatible Android Virtual Device.
3. Create three bot records and three independent writable data images.
4. Install the test APK and provision each bot separately.
5. Implement a single scroll-and-click task.
6. Add screenshot capture and an app-specific verification assertion.
7. Shut down and restart each bot to verify persistence.
8. Confirm that one bot cannot see another bot's app data or login session.

**Exit criteria:**

- Three bots have distinct state images.
- Each bot executes its task successfully.
- State survives shutdown and restart.
- No cross-bot session leakage is observed.
- Failures are recorded without silently losing state.

### Phase 2 — Ten-bot reliability test

**Objective:** Validate lifecycle handling and recovery.

Tasks:

1. Extend the registry to ten bots.
2. Add explicit boot and task timeouts.
3. Handle emulator process termination and port cleanup.
4. Add durable status transitions and error logging.
5. Simulate failures during boot and after an interaction.
6. Interrupt the runner and restart it.
7. Confirm that completed bots are not repeated automatically.
8. Measure task duration and disk consumption.

**Exit criteria:**

- The queue resumes safely after interruption.
- Uncertain outcomes are identified for review.
- Emulator shutdown is confirmed before the next task starts.
- Every completed task has a verifiable result.
- Storage growth is measured rather than estimated.

### Phase 3 — One-hundred-bot execution

**Objective:** Execute the complete local workload.

Tasks:

1. Provision 100 independent bot states.
2. Verify every registry record has a valid state path.
3. Confirm sufficient disk space before starting.
4. Run a small preflight subset.
5. Execute the remaining queue sequentially.
6. Save results and diagnostic evidence.
7. Generate a final summary of successes, failures, uncertain outcomes, duration, and storage usage.

**Exit criteria:**

- All 100 bots have terminal or explicitly reviewable statuses.
- No two bots share a writable data image.
- No two emulators run concurrently.
- No task is reported as successful without verification.
- The final report accounts for every bot.

## 7. Resource planning

Only one emulator runs at a time, so RAM requirements are governed primarily by the active emulator and macOS background processes.

Storage is the main scaling uncertainty.

| Average allocated state per bot | Storage for 100 bots |
|---:|---:|
| 100 MB | 10 GB |
| 250 MB | 25 GB |
| 500 MB | 50 GB |
| 1 GB | 100 GB |

These are planning scenarios, not measurements. Actual image allocation depends on the Android version, application data, installed packages, and image format.

Measure allocated disk usage after provisioning three bots and extrapolate before creating the remaining images.

Total execution time is approximately:

    total time = 100 × average time per bot

At 40 seconds per bot, the estimated total is about 67 minutes. Actual duration may be much longer if each bot needs first-boot initialization, app installation, login provisioning, or manual review.

## 8. Reliability and safety requirements

- Use explicit timeouts for emulator startup, UI readiness, and task execution.
- Check emulator process exit and device disconnection before reusing the port.
- Record an error and preserve the state image after a failure.
- Do not delete or recreate a failed bot's state automatically.
- Avoid automatic retries for actions with uncertain side effects.
- Verify UI state before and after each critical interaction.
- Keep credentials and session-bearing data out of Git.
- Use only authorized test accounts and an application environment where automation is permitted.
- Maintain a clear mapping from bot ID to state image and result record.

## 9. Repository structure

    android-bot-runner/
    ├── README.md
    ├── config.json
    ├── bots.json
    ├── runner.py
    ├── android.py
    ├── verification.py
    ├── database.py
    ├── app.apk
    ├── results.sqlite
    ├── state/
    ├── logs/
    ├── screenshots/
    └── tests/

Responsibilities:

- `runner.py`: scheduling, lifecycle, and recovery.
- `android.py`: emulator and ADB operations.
- `verification.py`: app-specific readiness and outcome assertions.
- `database.py`: SQLite schema and status transitions.
- `tests/`: unit tests and small integration tests for the runner.

## 10. Immediate next actions

1. Confirm the Android SDK and AVD are installed.
2. Identify the target app's package name and APK.
3. Create three independent state images.
4. Provision three test sessions.
5. Implement app-specific UI readiness and success assertions.
6. Run the three-bot pilot.
7. Inspect persistence, isolation, and recovery behavior.
8. Proceed to ten bots only after the pilot passes.

**Definition of done:** The system can run 100 independently provisioned, sequential Android tasks, preserve each bot's app state, recover safely from interruption, and provide an auditable outcome for every task.
