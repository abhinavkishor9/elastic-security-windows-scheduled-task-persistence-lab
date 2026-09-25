# Elastic Security Lab 06 — Scheduled Task Persistence

## Overview

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

## Objectives

- Understand Scheduled Tasks as a Windows persistence mechanism.
- Create a controlled and reversible Scheduled Task.
- Inspect task triggers, actions, user context, and state.
- Identify `schtasks.exe` activity in Elastic telemetry.
- Investigate Scheduled Task creation and manual execution.
- Correlate task activity with process telemetry.
- Document cases where expected process telemetry is not present.
- Distinguish controlled task activity from assumptions about execution.
- Remove the test Scheduled Task and verify remediation.
- Apply an evidence-based approach to persistence analysis.

## Scenario

A SOC analyst is investigating possible persistence through Windows Scheduled Tasks.

A controlled task named `ElasticLab06` is created using `schtasks.exe`. The task is configured with an `ONLOGON` trigger and an action that launches:

`C:\Windows\System32\notepad.exe`

The analyst validates the task locally and then uses Elastic to investigate the task creation and execution commands.

The analyst also checks whether the configured executable appears as an independent process event. The absence of a standalone `notepad.exe` result becomes an important telemetry limitation and is documented rather than inferred.

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

## Process Investigation Principle

The investigation follows:

```text
Task Creation
    |
    v
Task Name
    |
    v
Trigger
    |
    v
Action / Executable
    |
    v
Task Execution
    |
    v
Process Telemetry
    |
    v
Assessment
```

Each stage should be supported by direct evidence where possible.

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

## Final Assessment

The investigation successfully demonstrated Scheduled Task creation, configuration, execution commands, and remediation through both Windows and Elastic telemetry.

Elastic provided strong evidence for the `schtasks.exe` activity and the task's configured action. However, the available data did not provide a standalone `notepad.exe` process event, so direct confirmation of the final application process was not claimed.

The lab demonstrates how a SOC analyst should trace a persistence mechanism across multiple evidence sources while clearly separating **confirmed observations, correlation, and telemetry limitations**.
