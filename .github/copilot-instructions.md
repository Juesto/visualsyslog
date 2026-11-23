# Repository Instructions — Visual Syslog Modernization

## Project objective

Modernize the existing Visual Syslog Server into a native Windows application using ISO C++11.

Preserve existing functionality and behavior wherever practical. This is a port and architectural modernization, not a product redesign or feature-reduction exercise.

## Platform and compiler requirements

- Windows-only application. Linux and macOS are explicitly out of scope.
- ISO C++11; do not use later language features.
- MSVC is the primary compiler.
- GCC and Clang on Windows must remain viable alternatives.
- Use native Visual Studio solutions and project files as the primary build system.
- Do not introduce CMake.
- Prefer native Visual Studio toolsets and compiler selection where supported.

Compatibility targets:

| Windows version | Architecture |
|---|---|
| Windows XP SP3 | x86 |
| Windows Vista SP2 | x86/x64 |
| Windows 7 SP1 | x86/x64 |
| Windows 10 22H2 | x64 |
| Windows 11 | x64 |
| Windows 11 | ARM64 |

Treat older Windows compatibility as an explicit requirement to investigate and verify. Do not assume API availability based on the development machine's SDK.

Separate compile-time availability, runtime API availability and operating-system support. Newer functionality must not unnecessarily raise the minimum OS requirement for the entire application.

Native ARM64 support is preferred for Windows 11 ARM64. An x86 build under emulation may be considered an interim alternative.

## Architecture

### Shared logging engine

There must be one reusable logging engine implementation, independent of VCL, GUI code and hosting mechanisms.

The engine is responsible for:
- Network reception.
- Message parsing and processing.
- Filtering and routing.
- Log creation and writing.
- Rotation and retention.
- Automated alerts and actions.
- Operational configuration.

Do not duplicate engine functionality between standalone and service modes.

### Standalone mode

The GUI process can host the shared engine directly.

- No Windows service installation is required.
- The application must remain usable by standard users without administrative privileges, subject to normal Windows permissions.
- The GUI owns the engine, listeners and associated log storage.
- Configuration and storage should be scoped to the current user.

### Service mode

A Windows service hosts the same shared engine for machine-wide operation.

- The service owns listeners, processing, configuration, storage and rotation.
- Logging continues independently of interactive user sessions.
- The GUI connects as a reader and management client.
- Multiple authorized GUI instances can connect simultaneously.
- Configuration and storage are machine-wide.

### Resource ownership

Only one engine instance may own a particular set of listeners, configuration and storage at a time.

This applies to:
- Network listeners and bound ports.
- Active log files.
- Log rotation and retention.
- Automated processing and actions.
- Shared configuration state.

Standalone and service modes must not silently compete for the same resources.

Do not rely solely on port-binding failures or file-access errors to detect ownership conflicts. Establish explicit ownership and lifecycle coordination. If ownership cannot be obtained, report the reason clearly.

## GUI responsibilities

In service mode, the GUI is a reader and management client, not an independent logging engine.

Preserve existing user-facing functionality wherever practical:
- Live message display.
- Filtering and searching.
- Highlighting and presentation.
- Historical browsing.
- Loading rotated log files.
- Display preferences.
- Management operations supported through the service.

The GUI must not independently write, rotate or modify service-managed logs.

In standalone mode, the same GUI may host the shared engine and provide the existing application functionality.

User-session-dependent functionality, such as popup notifications and sounds, must be handled appropriately. Do not assume that a Windows service can interact directly with an interactive desktop.

## Storage and concurrency

Persistent log files are authoritative historical records.

- The active engine is the sole writer to its storage.
- GUI readers must not interfere with writing or rotation.
- Historical browsing must support rotated logs.
- Investigate Windows file-sharing semantics, rename behavior, handle lifetime and reopening after rotation.
- Do not assume that read-only file access automatically makes concurrent rotation safe.

Separate live message delivery from historical storage.

In service mode, IPC should deliver live messages to connected viewers. A reconnecting viewer should be able to recover missed messages using a reliable sequence or cursor mechanism. Do not rely on timestamps alone as unique message identifiers.

## Interprocess communication and security

Named pipes are the preferred initial IPC mechanism, subject to compatibility and security review.

Support:
- Multiple simultaneous clients.
- Client disconnection and reconnection.
- Versioned protocol messages.
- Message framing and size limits.
- Error reporting.
- Live message delivery.
- Configuration and management requests.
- Service lifecycle notifications.

Use appropriate Windows security descriptors and access checks.

Separate reader permissions from administrative and management permissions. Standard users should be able to read logs without automatically receiving administrative control over the service.

Do not trust clients merely because they connect through local named pipes.

## Dependency policy

Prefer Windows-provided facilities over third-party dependencies.

| Requirement | Preferred facility |
|---|---|
| Networking | Winsock2 |
| TCP/UDP | Winsock2 |
| TLS | SChannel |
| Cryptography | Windows Cryptography APIs |
| Service management | Service Control Manager |
| IPC | Named pipes |
| File access | Win32 file APIs |
| Registry access | Windows Registry APIs |
| GUI | Win32 and Common Controls |
| Synchronization | Windows synchronization APIs |

This is a preference, not permission to make unsuitable substitutions.

For each existing third-party dependency:
1. Identify its actual purpose and usage.
2. Determine whether Windows provides an adequate equivalent.
3. Identify behavioral and compatibility differences.
4. Recommend retaining, replacing or removing it.
5. Explain the trade-offs.

Do not introduce new third-party libraries without explicit approval.

Do not confuse build-time tools with application runtime dependencies.

## Preservation and code quality

Treat the original implementation as the functional reference.

Do not:
- Remove features because they are inconvenient to port.
- Simplify filtering or processing behavior without approval.
- Change log formats unnecessarily.
- Redesign the GUI without a requirement.
- Perform unrelated cleanup.
- Assume old code is defective simply because it is old.
- Assume documented behavior is implemented correctly without checking.

Distinguish between:
- Existing behavior.
- Existing defects.
- Required compatibility changes.
- Proposed improvements.

Code requirements:
- Clear, maintainable C++11.
- Explicit resource ownership and RAII.
- No undefined behavior.
- Check Windows API and Winsock return values.
- Handle errors explicitly; avoid silently suppressing failures.
- Avoid unnecessary global state.
- Keep GUI code out of the reusable engine.
- Keep hosting and service concerns out of core processing logic.
- Avoid compiler-specific extensions in shared code.

## Compatibility and testing

Do not claim compatibility or successful testing without evidence.

Consider:
- Compiler and language-standard compatibility.
- Windows API availability.
- Runtime and operating-system compatibility.
- Architecture-specific behavior.
- Dependencies and redistributables.

Tests should cover, where applicable:
- RFC 3164 and RFC 5424 parsing.
- UDP and TCP reception.
- Message framing and malformed input.
- Filtering and routing.
- Log writing and rotation.
- Historical browsing.
- Concurrent readers.
- Viewer disconnection and reconnection.
- Service startup and shutdown.
- Standalone startup and shutdown.
- Ownership conflicts.
- Standard-user permissions.
- Multiple simultaneous GUI clients.

Distinguish code that compiles from functionality that has actually been verified.

## Agent behavior and authorization

The agent is an implementation assistant, not the project architect.

- Inspect existing code before editing.
- Make small, reviewable changes.
- Explain significant architectural decisions.
- Do not perform wholesale rewrites.
- Do not change architecture, language standard, dependencies, compatibility targets or existing functionality without approval.
- Do not install dependencies or alter the development environment without approval.
- Do not modify user data or unrelated files.
- Do not create persistent progress reports, audit documents, roadmaps or tracking files unless explicitly requested.
- Do not maintain redundant project documentation.
- Do not assess AI authorship based on comment style or wording; focus on technical evidence.

When proposing a significant change, explain:
1. Existing situation.
2. Problem or limitation.
3. Proposed change.
4. Alternatives.
5. Compatibility implications.
6. Dependencies introduced or removed.
7. Verification required.

The user must approve significant architectural changes before implementation.
