# TestPilot — Agent Instructions

## Product goal

Build a standalone, reusable MCP server that lets an agent discover authorized .NET test projects, build them, discover and run tests, inspect structured results and logs, and cancel work. Development must also work locally with safe demonstration projects and no company code, data, or credentials.

## Initial compatibility target

- Windows 11 x64; implement the server in C# on .NET 10.
- Connect to Codex desktop through a local STDIO MCP server. Keep STDOUT exclusively for MCP protocol traffic; send server diagnostics to STDERR.
- Build projects with the installed .NET SDK selected by normal .NET rules, including `global.json`. Report detected SDK, runtime, and runner versions; claim compatibility only for versions actually validated.
- Initial test target: `net10.0`, native xUnit v3 runner 3.2.2. Do not silently switch to `dotnet test`, VSTest, or MTP.
- Use Debug by default unless a project profile or explicit request specifies another configuration.

## Design boundaries

- Separate the MCP host and typed/versioned tool contracts from job coordination, test-runner adapters, and process supervision. Keep public job and result models independent of xUnit XML and Windows APIs.
- Start with the native xUnit v3 runner on Windows. Add other runners and operating systems through explicit adapters; reject unsupported capabilities instead of silently approximating them.
- Accept project identifiers discovered under configured authorized roots, not arbitrary executable, shell, report, or config paths from the agent. Resolve real paths and enforce root boundaries, including symlinks and junctions.
- Use a user-provided config file via `--config`; do not edit Codex configuration or write configuration into project repositories automatically.
- Launch `dotnet` directly with argument lists, never through PowerShell or `cmd`. Resolve target frameworks and output paths from evaluated MSBuild properties rather than assuming `bin/Debug/net10.0`.
- Do not run tests if the current build fails, even if an older test assembly exists. Preserve project working directory, environment, SDK selection, build props, resources, and package configuration.

## Jobs, results, and logs

- Build, discovery, and run calls should return a job ID promptly; jobs remain queryable independently of the initiating call. Expose typed structured results, job state and phase, paginated diagnostics/logs/results, and cancellation.
- Run at most one operation per authorized root at a time; return a clear busy response rather than queueing work.
- Require an explicit `all` selection to run the full suite. Class, method, or discovered-ID filters must be exact; never broaden a failed selection silently. Reject stale discovered IDs.
- Use native xUnit discovery and its XML report as the structured source of run results. Distinguish discovered test definitions from executed cases (including theories). Preserve per-test duration, outcome, error, stack trace, skip reason, and output where available, as well as fixture/assembly infrastructure errors.
- A zero-test run, missing or invalid report, process error, or infrastructure failure must not be reported as success. Keep build/startup time separate from individual test durations.
- Capture stdout and stderr concurrently and persist them progressively. Keep logs and artifacts available after completion; support pagination without silently truncating large output. Treat artifact-write failures as job failures.
- Cancellation and timeouts apply to build, discovery, and execution. On Windows, manage each operation and its child processes with a Job Object; never use global .NET server shutdown commands. Do not mark a job cancelled or release its root lock while managed processes remain active.
- Persist enough state to mark unfinished work `interrupted` after server restart; do not automatically replay it. Preserve partial logs and mark results incomplete.

## Validation and security

- Validate the tool first with safe demonstration projects.
- Preserve the selected project's working directory, inherited environment, SDK selection, build properties, resources, and package configuration. Do not require changes to the project's tests or runtime configuration unless explicitly requested.
- Codex must be able to invoke test execution through the MCP without being granted general network access itself. Tests may need network access to perform their intended work.
- The MCP is not a sandbox. Built and executed code inherits the server process's filesystem, network, and credential access. Folder allow-lists restrict requested targets but cannot prevent build/test code or dependencies from accessing resources. Redaction does not guarantee confidentiality, and forced cancellation does not guarantee cleanup.
- Do not claim that the MCP can restrict test code to only the network destinations it needs. If that narrower network boundary is required, it needs a separate enforcement mechanism and must be designed explicitly.
- Report only logs emitted by the project; messages suppressed by its logger or application configuration cannot be reconstructed.

## Formatting

- Preserve Windows-style CRLF line endings when creating or editing files in this repository; do not normalize existing files to LF.

## Scope discipline

- Keep the first implementation focused on Windows x64, STDIO, and the native xUnit v3 runner. Treat Linux/macOS, VSTest/MTP, coordinated multi-project execution, dynamic plugins, interactive debugging, credential management, network restrictions, machine policy changes, SDK installation, Git changes, and self-updating as out of scope unless explicitly brought back into scope.
- Do not install software, publish artifacts, alter machine or Codex settings, or modify Git history without explicit authorization.
- Communicate with the user in European Portuguese unless asked otherwise.
