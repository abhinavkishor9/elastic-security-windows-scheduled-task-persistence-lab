# Investigation Notes 

## Agent Validation

Fleet showed:

```text
Status: Healthy
Host: DESKTOP-9MMM37V
Policy: Windows-SOC-Lab
Agent Version: 9.5.4
```

This confirmed that the Elastic Agent was active during the investigation.

## Task Creation

The following command was executed:

```powershell
schtasks.exe /Create /SC ONLOGON /TN "ElasticLab06" /TR "C:\Windows\System32\notepad.exe" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" has successfully been created.
```

## Task Validation

The task was checked using:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

Observed:

```text
TaskName: ElasticLab06
State: Ready
```

The task configuration was also inspected:

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

## Detailed Task Metadata

The following command was used:

```powershell
schtasks.exe /Query /TN "ElasticLab06" /V /FO LIST
```

Observed:

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

These fields established the intended trigger, action, and user context.

## Manual Execution

The task was manually invoked:

```powershell
schtasks.exe /Run /TN "ElasticLab06"
```

Windows returned:

```text
SUCCESS: Attempted to run the scheduled task "ElasticLab06".
```

The task was later observed in:

```text
State: Queued
```

The detailed query showed:

```text
Last Run Time: 25-09-2026 06:37:08
Last Result: 0
```

The task still referenced:

```text
C:\Windows\System32\notepad.exe
```

## Elastic Process Investigation

The following query searched for task-related command lines:

```esql
FROM logs-*
| WHERE process.command_line LIKE "*ElasticLab06*"
| KEEP @timestamp, host.name, user.name, process.name, process.pid, process.parent.name, process.parent.pid, process.command_line, process.executable
| SORT @timestamp DESC
```

The query returned multiple `schtasks.exe` events.

### Task Creation Event

Observed:

```text
Timestamp: Sep 25, 2026 @ 06:30:43.336
Process: schtasks.exe
PID: 13840
Parent: pwsh.exe
Parent PID: 10096
User: Dell
```

Command line:

```text
"C:\Windows\System32\schtasks.exe" /Create /SC ONLOGON /TN "ElasticLab06" /TR "C:\Windows\System32\notepad.exe" /F
```

### Task Run Event

Observed:

```text
Timestamp: Sep 25, 2026 @ 06:37:08.402
Process: schtasks.exe
PID: 21304
Parent: pwsh.exe
Parent PID: 10096
User: Dell
```

Command line:

```text
"C:\Windows\System32\schtasks.exe" /Run /TN ElasticLab06
```

### Task Query Event

Observed:

```text
Timestamp: Sep 25, 2026 @ 06:42:32.523
Process: schtasks.exe
PID: 25196
Parent: pwsh.exe
Parent PID: 10096
User: Dell
```

Command line included:

```text
schtasks.exe /Query /TN ElasticLab06 /V
```

## Null Process Metadata

Several additional `schtasks.exe` events were returned with some fields unset.

Examples included:

```text
PID: null
Parent: null
Command Line: null
```

while the executable path remained:

```text
C:\Windows\System32\schtasks.exe
```

These were documented as incomplete process metadata rather than interpreted as separate malicious activity.

## Notepad Investigation

The following query was executed:

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

Therefore, no standalone `notepad.exe` endpoint event was observed in the queried dataset.

## Evidence Correlation

The available evidence establishes:

```text
06:30:43
schtasks.exe /Create
        |
        v
ElasticLab06
        |
        +-- ONLOGON
        +-- C:\Windows\System32\notepad.exe
        |
        v
06:37:08
schtasks.exe /Run
        |
        v
Task state: Queued
        |
        v
Last Result: 0
```

The configuration and Scheduler state provide evidence that the task was created and processed.

The available Elastic data does not independently demonstrate the final `notepad.exe` process event.

## Remediation

The task was deleted using:

```powershell
schtasks.exe /Delete /TN "ElasticLab06" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" was successfully deleted.
```

Post-deletion validation:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

returned no matching task.

A second validation:

```powershell
schtasks.exe /Query /TN "ElasticLab06"
```

returned:

```text
ERROR: The system cannot find the file specified.
```

## Analyst Assessment

### Observed

- Scheduled Task creation.
- `ONLOGON` trigger.
- `notepad.exe` configured as the task action.
- `schtasks.exe /Create` telemetry.
- `schtasks.exe /Run` telemetry.
- `schtasks.exe /Query` telemetry.
- Task state transitions from `Ready` to `Queued`.
- Successful last result value of `0`.
- Incomplete metadata in some process events.
- No standalone `notepad.exe` event.

### Confirmed

- The controlled Scheduled Task existed.
- The intended executable was configured.
- The task was manually invoked.
- Elastic captured the relevant `schtasks.exe` commands.
- The test task was deleted successfully.

### Unknown

- Whether the specific manual `/Run` invocation resulted in an independently captured `notepad.exe` process event.
- Why the expected `notepad.exe` process event was absent from the queried Elastic dataset.

## Malicious Activity Assessment

The controlled task does not establish malicious persistence.

No evidence was demonstrated for:

- Malware execution
- Credential access
- Privilege escalation
- Command-and-control
- Defense evasion
- Confirmed endpoint compromise

The task was intentionally created for this investigation and referenced a legitimate Windows executable.

