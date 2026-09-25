# Troubleshooting Notes 

## Issue 1 — Initial Scheduled Task Search Returned No Existing Task

Before creating the test task, the following command was used:

```powershell
Get-ScheduledTask | Where-Object {$_.TaskName -eq "ElasticLab06"}
```

No matching task was returned.

This established that the controlled task did not already exist before creation.

### Lesson

Always establish a baseline before creating a persistence artifact.

---

## Issue 2 — Task State Changed from `Ready` to `Queued`

Immediately after creation, the task showed:

```text
State: Ready
```

After running:

```powershell
schtasks.exe /Run /TN "ElasticLab06"
```

the task later showed:

```text
State: Queued
```

The detailed task query also showed:

```text
Last Run Time: 25-09-2026 06:37:08
Last Result: 0
```

### Interpretation

The state change was treated as Scheduler state information.

It was not interpreted as proof of a specific application process because no standalone `notepad.exe` event was found in Elastic.

### Lesson

Task state and process execution are related but should be investigated separately.

---

## Issue 3 — `notepad.exe` Query Returned No Results

### Query

```esql
FROM logs-*
| WHERE process.name == "notepad.exe"
| KEEP @timestamp, host.name, user.name, process.name, process.pid, process.parent.name, process.parent.pid, process.command_line, process.executable
| SORT @timestamp DESC
```

### Result

```text
No results match your search criteria
```

### Investigation

The Scheduled Task clearly referenced:

```text
C:\Windows\System32\notepad.exe
```

and the task was manually invoked.

However, the queried Elastic dataset did not return a standalone `notepad.exe` process event.

### Resolution

The investigation documented the result as a telemetry limitation.

It did not claim:

```text
notepad.exe definitely executed
```

based solely on the task configuration.

### Lesson

A configured task action is not equivalent to a directly observed process event.

---

## Issue 4 — Searching the Task Name Returned `schtasks.exe`, Not `notepad.exe`

### Query

```esql
FROM logs-*
| WHERE process.command_line LIKE "*ElasticLab06*"
| KEEP @timestamp, host.name, user.name, process.name, process.pid, process.parent.name, process.parent.pid, process.command_line, process.executable
| SORT @timestamp DESC
```

### Result

The query returned multiple:

```text
schtasks.exe
```

events.

These included:

```text
/Create
/Run
/Query
```

### Lesson

The task name is directly present in the `schtasks.exe` command line, so task-name hunting is effective for identifying task-management activity.

It does not automatically identify the process launched by the task.

---

## Issue 5 — Some `schtasks.exe` Events Had Null Metadata

Several returned events contained:

```text
PID: null
Parent: null
Command Line: null
```

while still showing:

```text
process.executable:
C:\Windows\System32\schtasks.exe
```

### Handling

These events were retained as telemetry observations but were not assigned additional meaning without supporting evidence.

Other events contained complete process context, including:

```text
PID
Parent Process
Parent PID
Command Line
User
```

### Lesson

Endpoint telemetry can contain partially populated process records. Correlate complete events where possible and document incomplete metadata rather than filling in missing values.

---

## Issue 6 — Determining the Parent of the Scheduled Task

The controlled `schtasks.exe` commands showed:

```text
Parent: pwsh.exe
```

with:

```text
Parent PID: 10096
```

This is the parent of the `schtasks.exe` process used to create or run the task.

It should not automatically be treated as the parent of the eventual scheduled application.

### Important Distinction

```text
pwsh.exe
    |
    +-- schtasks.exe
```

is the observed task-management process relationship.

The eventual task-launched process may have a different execution context and should be validated independently.

### Lesson

Do not confuse the process that manages a Scheduled Task with the process that the task eventually launches.

---

## Issue 7 — Time Range

The investigation used:

```text
Last 15 minutes
```

The main controlled timestamps were:

```text
06:30:43.336
06:37:08.402
06:42:32.523
```

### Lesson

The selected time range must include task creation, execution, and validation events.

For controlled testing, a short window is useful because it reduces unrelated endpoint activity.

---

## Issue 8 — Task Deletion Verification

The task was removed with:

```powershell
schtasks.exe /Delete /TN "ElasticLab06" /F
```

Windows returned:

```text
SUCCESS: The scheduled task "ElasticLab06" was successfully deleted.
```

The PowerShell check:

```powershell
Get-ScheduledTask -TaskName "ElasticLab06"
```

returned no matching task.

The `schtasks.exe` check:

```powershell
schtasks.exe /Query /TN "ElasticLab06"
```

returned:

```text
ERROR: The system cannot find the file specified.
```

### Lesson

Use more than one verification method when practical.

---

