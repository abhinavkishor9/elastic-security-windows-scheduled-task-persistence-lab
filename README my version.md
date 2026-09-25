# elastic-security-windows-scheduled-task-persistence-lab
## Overview
Windows Task Scheduler allows programs or scripts to execute automatically according to triggers such as:

-  User logon
- System startup
- A specific time
- Other scheduled conditions

Scheduled Tasks are legitimate Windows functionality, so the existence of a scheduled task is not automatically malicious.

The investigation should follow:

Task Creation
    ↓
Task Name
    ↓
Trigger
    ↓
Action
    ↓
Command / Executable
    ↓
Task Context
    ↓
Process Execution
    ↓
Parent Process
    ↓
Assessment

This lab investigates **Windows Scheduled Task persistence** using Elastic Security and Elastic Defend.

Windows Task Scheduler is a legitimate Windows capability that can launch programs automatically based on triggers such as user logon. Because Scheduled Tasks are commonly used by both legitimate software and attackers, the presence of a task alone does not establish malicious persistence.

The investigation focuses on task creation, task configuration, execution activity, command-line evidence, and process telemetry.

A controlled Scheduled Task named `ElasticLab06` was created to launch:

`C:\Windows\System32\notepad.exe`

The task was then queried, manually triggered, investigated in Elastic, and finally removed.

## Lab Environment

| Component | Details |
|---|---|
| Endpoint | Windows 10 Pro 22H2 |
| Host | `DESKTOP-9MMM37V` |
| User | `Dell` |
| Elastic Platform | Elastic Security Serverless |
| Endpoint Integration | Elastic Defend |
| Elastic Agent | `9.5.4` |
| Agent Policy | `Windows-SOC-Lab` |
| Investigation Interface | Discover / ES|QL |
| Shell | PowerShell 7.6.6 |
| Time Range | Last 15 minutes |

## Lab Objectives

- Examine how a Windows Scheduled Task can maintain automatic execution through a defined trigger.
- Build a controlled Scheduled Task and inspect its configuration from the Windows host.
- Trace the lifecycle of a task from creation through manual invocation, queued state, and removal.
- Use `schtasks.exe` telemetry to identify administrative actions performed against the test task.
- Compare task configuration evidence with process-level telemetry to determine how far the execution chain can be observed.
- Analyze the relationship between the task-management process and the process eventually configured as the task action.
- Investigate incomplete `schtasks.exe` records and determine how missing process fields affect analysis.
- Evaluate the significance of task states such as `Ready` and `Queued` during an investigation.
- Distinguish evidence showing that a task was configured or invoked from evidence proving that its target application executed.
- Practice documenting gaps between expected behavior and available endpoint telemetry.
- Validate that the controlled persistence artifact can be completely removed after testing.
- Develop an investigation approach that separates **task configuration, execution evidence, process evidence, and analyst interpretation**.
- Identify the limits of process-based hunting when the expected child application is not present in the queried telemetry.
- Map the observed behavior to the appropriate persistence technique while keeping the assessment tied to the evidence actually collected.

## Lab Scenario

A SOC analyst is investigating a Windows endpoint for potential **persistence through Scheduled Tasks**. The analyst needs to determine whether a newly created task represents a legitimate administrative configuration or an artifact that requires further investigation.

A controlled Scheduled Task named `ElasticLab06` is created on the endpoint with an `ONLOGON` trigger. Its configured action points to:

```text
C:\Windows\System32\notepad.exe
```

The task is then inspected locally to determine its state, trigger, action, author, and execution context. It is also manually invoked so that the analyst can observe how the task behaves and what endpoint telemetry is generated.

The investigation focuses on:

- Task creation and configuration
- Trigger and execution context
- `schtasks.exe` command-line activity
- Parent-child relationships associated with task management
- Task states such as `Ready` and `Queued`
- Last run time and result
- Whether the configured executable appears as a separate process event
- Differences between confirmed telemetry and expected task behavior
- Removal and verification of the controlled persistence artifact

Elastic telemetry captures the `schtasks.exe` commands used to create, run, and query the task. However, the expected `notepad.exe` process event is not returned in the investigated dataset, creating an important distinction between **task configuration, task invocation, and independently observed process execution**.

The activity is intentionally benign and reversible. The test task is removed after the investigation and the deletion is verified.

The scenario is designed to demonstrate how a SOC analyst investigates Scheduled Task persistence by correlating **task configuration, command-line evidence, execution state, process telemetry, and telemetry limitations** without making assumptions beyond the available evidence.

## Task Creation

The following command was used:

```powershell
schtasks.exe /Create /SC ONLOGON /TN "ElasticLab06" /TR "C:\Windows\System32\notepad.exe" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" has successfully been created.
```

## Task Validation

The task was verified with:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

Observed:

```text
TaskName: ElasticLab06
State: Ready
```

The task configuration was then inspected:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06" | Select-Object TaskName, State, Actions, Triggers
```

Observed:

```text
TaskName: ElasticLab06
State: Ready
Actions: MSFT_TaskExecAction
Triggers: MSFT_TaskLogonTrigger
```

## Detailed Task Configuration

The task was queried using:

```powershell
schtasks.exe /Query /TN "ElasticLab06" /V /FO LIST
```

Important observed values included:

```text
HostName: DESKTOP-9MMM37V
TaskName: \ElasticLab06
Status: Ready
Logon Mode: Interactive only
Author: DESKTOP-9MMM37V\Dell
Task To Run: C:\Windows\System32\notepad.exe
Scheduled Task State: Enabled
Run As User: Dell
Schedule Type: At logon time
```

These values confirmed that the task was configured as an enabled logon-triggered task running under the `Dell` user context.

## Manual Task Execution

The task was manually triggered with:

```powershell
schtasks.exe /Run /TN "ElasticLab06"
```

Windows returned:

```text
SUCCESS: Attempted to run the scheduled task "ElasticLab06".
```

The task was subsequently observed in a `Queued` state.

A later detailed query showed:

```text
Status: Queued
Last Run Time: 25-09-2026 06:37:08
Last Result: 0
Task To Run: C:\Windows\System32\notepad.exe
Run As User: Dell
```

The successful result code and updated last-run time provided evidence that the task had been processed by Task Scheduler.

## Elastic Telemetry

The following ES|QL query was used:

```esql
FROM logs-*
| WHERE process.command_line LIKE "*ElasticLab06*"
| KEEP @timestamp, host.name, user.name, process.name, process.pid, process.parent.name, process.parent.pid, process.command_line, process.executable
| SORT @timestamp DESC
```

Elastic returned multiple `schtasks.exe` events.

Important controlled events included:

```text
Sep 25, 2026 @ 06:30:43.336
schtasks.exe
Parent: pwsh.exe
Parent PID: 10096
PID: 13840
Command: "C:\Windows\System32\schtasks.exe" /Create /SC ONLOGON /TN "ElasticLab06" /TR "C:\Windows\System32\notepad.exe" /F
```

```text
Sep 25, 2026 @ 06:37:08.402
schtasks.exe
Parent: pwsh.exe
Parent PID: 10096
PID: 21304
Command: "C:\Windows\System32\schtasks.exe" /Run /TN ElasticLab06
```

A later query event was also observed:

```text
Sep 25, 2026 @ 06:42:32.523
schtasks.exe
Parent: pwsh.exe
Parent PID: 10096
PID: 25196
Command: "C:\Windows\System32\schtasks.exe" /Query /TN ElasticLab06 /V ...
```

## Notepad Process Investigation

A dedicated process query was used:

```esql
FROM logs-*
| WHERE process.name == "notepad.exe"
| KEEP @timestamp, host.name, user.name, process.name, process.pid, process.parent.name, process.parent.pid, process.command_line, process.executable
| SORT @timestamp DESC
```

Result:

```text
No results match your search criteria
```

Therefore, the investigation does not claim that a standalone `notepad.exe` process event was captured by the queried Elastic dataset.

The task configuration clearly referenced `notepad.exe`, but the available telemetry did not independently show the resulting process.

## Key Findings

### Observed

- `ElasticLab06` was successfully created.
- The task used an `ONLOGON` trigger.
- The task action referenced `C:\Windows\System32\notepad.exe`.
- The task ran under the `Dell` user context.
- `schtasks.exe /Create` was captured by Elastic.
- `schtasks.exe /Run` was captured by Elastic.
- The task later showed a `Queued` state.
- The recorded last run time was `25-09-2026 06:37:08`.
- The last result was `0`.
- A standalone `notepad.exe` event was not returned by the process query.
- Some `schtasks.exe` telemetry contained null PID, parent, and command-line fields.

### Confirmed

- The Scheduled Task existed.
- The task configuration referenced the intended executable.
- The task was intentionally created for the lab.
- Elastic captured the key `schtasks.exe` commands.
- The test task was successfully deleted.
- Post-deletion checks confirmed that `ElasticLab06` no longer existed.

### Not Demonstrated

- Malicious Scheduled Task persistence
- Malware execution
- Credential theft
- Privilege escalation
- Command-and-control
- Defense evasion
- Confirmed endpoint compromise

## Telemetry Limitation

The most important limitation in this lab was the absence of a standalone `notepad.exe` process event.

The task was configured to launch:

`C:\Windows\System32\notepad.exe`

and the task showed a successful last result after the manual run. However, the dedicated Elastic process query returned no `notepad.exe` documents.

The investigation therefore records the task configuration and execution telemetry without claiming that the final application process was independently captured.

## MITRE ATT&CK

### T1053.005 — Scheduled Task/Job: Scheduled Task

The controlled activity demonstrates Windows Scheduled Task creation and execution.

The ATT&CK mapping describes the persistence mechanism used in the lab and does not indicate that the controlled task itself was malicious.

## Remediation

The test task was deleted using:

```powershell
schtasks.exe /Delete /TN "ElasticLab06" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" was successfully deleted.
```

The deletion was validated with:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

which returned no matching task.

A second check with:

```powershell
schtasks.exe /Query /TN "ElasticLab06"
```

returned:

```text
ERROR: The system cannot find the file specified.
```

This confirmed the test task was removed.

