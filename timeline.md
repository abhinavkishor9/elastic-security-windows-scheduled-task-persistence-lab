# Timeline — Scheduled Task Persistence

## Investigation Timeline

| Time | Activity | Process / Artifact | Evidence / Notes |
|---|---|---|---|
| Initial validation | Confirmed task did not already exist | `ElasticLab06` | No matching task found before controlled creation |
| 06:30:43.336 | Scheduled Task created | `schtasks.exe` PID `13840` | Parent `pwsh.exe`, `/Create`, `ONLOGON`, action `notepad.exe` |
| 06:32:46.875 | Task queried | `schtasks.exe` PID `25196` | `/Query /TN ElasticLab06 /V` |
| 06:32:46.908 | Related task-management event | `schtasks.exe` PID `25196` | Additional event with incomplete parent/command-line metadata |
| 06:37:08.402 | Scheduled Task manually triggered | `schtasks.exe` PID `21304` | `/Run /TN ElasticLab06`, parent `pwsh.exe` |
| 06:37:08.435 | Related task-management event | `schtasks.exe` | Additional event with incomplete process metadata |
| 06:37:08 | Task execution state updated | `ElasticLab06` | Last Run Time recorded as `25-09-2026 06:37:08`, Last Result `0` |
| 06:42:32.523 | Task queried again | `schtasks.exe` PID `25196` | `/Query /TN ElasticLab06 /V` |
| 06:42:32.556 | Related task-management event | `schtasks.exe` | Additional event with incomplete metadata |
| Investigation | Notepad process hunt | `notepad.exe` | No standalone `notepad.exe` event returned |
| Remediation | Scheduled Task deleted | `ElasticLab06` | Deletion succeeded |
| Post-remediation | Task verification | `ElasticLab06` | `Get-ScheduledTask` and `schtasks /Query` confirmed absence |

## Initial Baseline

Before creation, the following command was used:

```powershell
Get-ScheduledTask | Where-Object {$_.TaskName -eq "ElasticLab06"}
```

No matching task was returned.

This established that `ElasticLab06` was not already present before the controlled activity.

## 06:30:43 — Scheduled Task Creation

Elastic recorded:

```text
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

This established the controlled Scheduled Task creation event.

## 06:32:46 — Task Query

Elastic recorded:

```text
schtasks.exe
PID: 25196
Parent: pwsh.exe
Parent PID: 10096
```

Command line included:

```text
schtasks.exe /Query /TN ElasticLab06 /V
```

This was an administrative inspection of the task rather than creation of a new persistence artifact.

## 06:37:08 — Manual Task Execution

The task was manually invoked with:

```powershell
schtasks.exe /Run /TN "ElasticLab06"
```

Elastic recorded:

```text
Process: schtasks.exe
PID: 21304
Parent: pwsh.exe
Parent PID: 10096
User: Dell
```

The task later showed:

```text
State: Queued
Last Run Time: 25-09-2026 06:37:08
Last Result: 0
```

## 06:42:32 — Task Validation

The task was queried again using:

```text
schtasks.exe /Query /TN ElasticLab06 /V
```

The query event was visible in Elastic with:

```text
PID: 25196
Parent: pwsh.exe
Parent PID: 10096
User: Dell
```

## Notepad Process Investigation

The following query was used:

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

The task configuration referenced:

```text
C:\Windows\System32\notepad.exe
```

but the available Elastic query did not return a standalone Notepad process event.

This was recorded as a telemetry limitation.

## Process Metadata Anomalies

Some `schtasks.exe` records contained:

```text
PID: null
Parent: null
Command Line: null
```

while retaining the executable path:

```text
C:\Windows\System32\schtasks.exe
```

These records were documented as incomplete telemetry and were not independently interpreted as malicious.

## Remediation

The task was deleted using:

```powershell
schtasks.exe /Delete /TN "ElasticLab06" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" was successfully deleted.
```

## Post-Deletion Validation

The following command returned no matching task:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

A second validation:

```powershell
schtasks.exe /Query /TN "ElasticLab06"
```

returned:

```text
ERROR: The system cannot find the file specified.
```

This confirmed that the controlled Scheduled Task was removed.

## Final Timeline Assessment

```text
Baseline check
      ↓
Scheduled Task created
      ↓
Task configuration validated
      ↓
schtasks.exe /Run executed
      ↓
Task state / last run reviewed
      ↓
Notepad process hunt performed
      ↓
Missing process telemetry documented
      ↓
Task deleted
      ↓
Deletion verified
```

## Final Assessment

The investigation confirmed the creation, configuration, management, and removal of the controlled Scheduled Task.

Elastic provided strong evidence for the `schtasks.exe` commands and task-management activity.

The absence of a standalone `notepad.exe` event was documented as a telemetry limitation. No malicious persistence, malware execution, credential access, privilege escalation, command-and-control, or confirmed compromise was demonstrated.
