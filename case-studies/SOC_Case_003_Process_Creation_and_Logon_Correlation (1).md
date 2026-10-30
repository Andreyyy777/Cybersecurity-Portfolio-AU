# Case 003 — Windows Process Creation and Logon Correlation

**Author:** Andrey Kurbanov  
**Investigation date:** 30 October 2026  
**Environment:** Windows 11 ARM64 virtual machine running in UTM on macOS  
**Classification:** Controlled home-lab exercise  
**Outcome:** Process auditing and command-line visibility validated; selected process activity correlated with a successful logon.

## Executive summary

This exercise investigated how Windows Security events can establish which account session launched a program, which process created it, and which arguments were supplied. I enabled process-creation auditing, generated known activity, identified missing command-line telemetry, enabled command-line inclusion, and repeated controlled tests.

The evidence contains one successful-logon event (4624) and seven process-creation events (4688). All seven process events share the creator logon identifier associated with the selected successful logon. Parent and child process identifiers establish the tested PowerShell-to-command-prompt-to-whoami chains. Later events contain command arguments that were absent from the earlier Notepad records.

The exported EVTX was independently parsed and its SHA-256 digest matched the accompanying checksum file. The observed activity is consistent with the documented lab actions. This exercise does not represent a discovered compromise or an assessment of the entire host.

## Objective and scope

- Validate that successful process creation is recorded in the Security log.
- Identify and address a command-line visibility gap.
- Correlate successful authentication with subsequent process activity.
- Reconstruct parent–child relationships using event fields and timestamps.
- Preserve a small, reproducible evidence set and verify its integrity after transfer.

All actions were performed in my own Windows virtual machine. The exported evidence is a deliberately selected subset of the Security log, not a complete acquisition. The public report omits the machine-specific account SID and hostname; the original evidence retains them privately.

## Tools and method

Windows PowerShell, `auditpol`, `reg`, `Get-WinEvent`, `wevtutil`, `Get-FileHash`, and `Compress-Archive` were used for configuration, collection and preservation. Windows Event Viewer and PowerShell output supported inspection. An independent review used Python EVTX parsing to read the uploaded binary log and compared its fields with the text exports.

### 1. Establish the audit baseline

The initial Process Creation audit-policy check showed **No Auditing**. Successful process-creation auditing was enabled:

```powershell
auditpol /get /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}"
auditpol /set /subcategory:"{0CCE922B-69AE-11D9-BED3-505054503030}" /success:enable
```

Subsequent 4688 events demonstrated that process-creation telemetry was being collected. The initial setting is documented by the lab screenshot; that screenshot and a policy-change event are not included in the five-file evidence archive.

### 2. Generate and inspect known process activity

```powershell
$testStart = Get-Date
Start-Process notepad.exe
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4688
    StartTime = $testStart
} | Where-Object { $_.Message -match 'notepad\.exe' } |
    Format-List TimeCreated, Id, RecordId, Message
```

Records 23300 and 23301 showed PowerShell creating a Notepad process, followed by that Notepad process creating another Notepad process. Both command-line fields were empty. The reason for the second Notepad process was not established by this exercise.

### 3. Correlate the account session

The process creator logon identifier `0xe4195` was matched to the target logon identifier in successful-logon record 22764. The account SID and host also matched across the selected records.

The logon record reports type 2 (interactive), source address `127.0.0.1`, and `svchost.exe` in its process field. The address is loopback, not an external source. The process field in this logon event is separate from the creator-process fields used to reconstruct later process chains.

This correlation establishes the account-session context recorded by Windows. It does not independently prove the identity of the person operating the account. Logon identifiers must be interpreted within the relevant host and boot context.

### 4. Address missing command-line telemetry

The registry query returned that the specified key or value could not be found:

```powershell
reg query "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled
```

Command-line inclusion was enabled in the lab:

```powershell
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1 /f
```

The original empty command-line fields were not retroactively populated. New controlled activity was required to validate the change. Command-line logging can capture sensitive arguments; test commands contained no passwords or tokens.

### 5. Validate the change with controlled commands

```powershell
& "$env:SystemRoot\System32\cmd.exe" /d /c "echo SOC_LAB_TEST_001"
& "$env:SystemRoot\System32\cmd.exe" /d /c "whoami"
& "$env:SystemRoot\System32\cmd.exe" /d /c "whoami /user"
```

The marker test produced record 23613 with the full command line. The two whoami tests each produced a command-prompt process and a child whoami process. The recorded arguments distinguish the plain account-name query from the query requesting the account SID.

## Evidence timeline

Times below are 30 October 2026, displayed in the lab's local time (UTC+11). Binary EVTX timestamps were checked in UTC; for example, local 11:48:21 corresponds to 00:48:21 UTC. Times are shown to the second for readability; the EVTX retains finer precision.

| Local time | Record ID | Event ID | Observed activity |
|---|---:|---:|---|
| 11:48:21 | 22764 | 4624 | Successful interactive logon; target logon ID `0xe4195` |
| 12:03:38 | 23300 | 4688 | PowerShell `0x1834` creates Notepad `0x2580` |
| 12:03:40 | 23301 | 4688 | Notepad `0x2580` creates Notepad `0x219c` |
| 12:47:42 | 23613 | 4688 | PowerShell `0x1834` creates cmd `0x152c`; marker command recorded |
| 12:51:19 | 23617 | 4688 | PowerShell `0x1834` creates cmd `0x1248`; arguments include `whoami` |
| 12:51:19 | 23618 | 4688 | cmd `0x1248` creates whoami `0x1680` |
| 12:58:04 | 23694 | 4688 | PowerShell `0x1834` creates cmd `0x2f78`; arguments include `whoami /user` |
| 12:58:04 | 23695 | 4688 | cmd `0x2f78` creates whoami `0x27f0`; `/user` recorded |

## Analysis and findings

**Session correlation:** All seven 4688 records contain creator logon ID `0xe4195`, matching the target logon ID of record 22764. The first observed Notepad launch occurred approximately 15 minutes 17 seconds after that logon.

**Process relationships:** In the first whoami test, the new process ID in record 23617 (`0x1248`) matches the creator process ID in record 23618. The second test repeats the relationship with `0x2f78` in records 23694 and 23695. Matching host, timestamps and executable paths strengthen these associations. Process IDs are reusable and should not be treated as permanent identifiers.

**Telemetry improvement:** Both earlier Notepad records have empty command-line fields. All five later process records contain command-line data. This before-and-after result supports the effectiveness of the lab configuration change for the tested processes; it does not establish universal coverage of all execution mechanisms.

**Privilege context:** The selected process records report `TokenElevationTypeDefault` and mandatory label `S-1-16-12288`. The logon record reports an elevated token. These fields are retained as context; this exercise did not independently assess the complete UAC configuration or demonstrate privilege escalation.

**Disposition:** Expected activity in a controlled exercise. Running whoami alone is insufficient evidence of malicious discovery. Interpretation depends on surrounding activity and the known test context.

## Evidence preservation and verification

The following five files were collected into `SOC_Process_Lab_20261030_131128.zip`:

| File | Purpose |
|---|---|
| `SOC_Process_Test_2026-10-30.txt` | Human-readable Notepad records 23300 and 23301 |
| `SOC_Logon_Process_2026-10-30.txt` | Logon record 22764 and both Notepad records |
| `SOC_Whoami_Comparison_2026-10-30.txt` | Records 23617, 23618, 23694 and 23695 |
| `SOC_Process_Evidence_20261030_130340.evtx` | Eight selected native Security events |
| `SOC_Process_Evidence_20261030_130340.evtx.sha256.csv` | SHA-256 value and original export path |

The record selections in the text reports overlap. They must not be counted as additional unique events. Record 23613 is present in the EVTX even though it is not included in the three text reports.

The native export used this selection:

```powershell
$evtx = Join-Path ([Environment]::GetFolderPath('Desktop')) (
    'SOC_Process_Evidence_{0}.evtx' -f (Get-Date -Format 'yyyyMMdd_HHmmss')
)
wevtutil epl Security "$evtx" /q:"*[System[(EventRecordID=22764 or EventRecordID=23300 or EventRecordID=23301 or EventRecordID=23613 or EventRecordID=23617 or EventRecordID=23618 or EventRecordID=23694 or EventRecordID=23695)]]"
if ($LASTEXITCODE -ne 0) { throw 'Event export failed.' }
Get-FileHash -LiteralPath $evtx -Algorithm SHA256 |
    Export-Csv -LiteralPath "$evtx.sha256.csv" -NoTypeInformation -Encoding UTF8
```

These record IDs identify this acquisition only; another run will have different IDs.

The ZIP was copied from the Windows Desktop to the UTM shared folder. Source and destination ZIP hashes matched in the lab. After upload, review independently confirmed:

- ZIP CRC checks passed for all five entries.
- The EVTX parsed successfully and contained exactly eight records: one 4624 and seven 4688.
- Record IDs and relevant fields agreed with the text exports, accounting for hexadecimal zero-padding and timestamp representation.
- The 69,632-byte EVTX matched the SHA-256 recorded in the CSV:

```text
7AC4137E1519F0F82087CE134F6F782E75BD11E3AF121B33A312452355B6F694
```

Matching hashes demonstrate byte consistency with the supplied checksum. They do not independently establish source authenticity or a forensic chain of custody. Raw logs and machine-specific identifiers are retained privately rather than included in this public write-up.

## Limitations

- The export contains selected records, not all events in the time window.
- Event 4688 records process creation and, when available, arguments. It does not record command output or prove successful command completion.
- Earlier missing command-line data cannot be reconstructed from these events.
- The creation event for the parent PowerShell process is not in this acquisition; its identity is recorded in the child events.
- The reason for Notepad creating a second process remains unverified.
- No file-signature, malware, network-traffic or full endpoint investigation was performed.
- The tested configuration is suitable for learning; production changes require the organisation's policy, retention and access-control requirements.

## Skills demonstrated and next development

This case demonstrates audit-policy inspection, PowerShell event filtering, structured event-field analysis, logon correlation, process-tree reconstruction, command-line telemetry validation, targeted EVTX export and SHA-256 verification.

The next development step is to ingest the same types of events into a SIEM and implement a small detection exercise with both expected and unexpected parent-process contexts. The detection should retain the user, host, timestamp, parent process and command line so an analyst can explain an alert rather than treating a command name alone as proof of an incident.

## Conclusion

The lab established a verified evidence trail from successful logon to process creation and showed how enabling command-line inclusion improves investigative visibility. Both whoami execution chains were reconstructed from the exported events. The exercise was completed with a preserved, hash-checked evidence set and a clearly bounded conclusion: the selected records match the controlled actions performed in the lab.
