# Timeline 

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

