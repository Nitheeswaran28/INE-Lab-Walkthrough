# Kibana: Windows Event Logs I source notes

The INE/AttackDefense PDF, walkthrough 1182, supplies the five investigations. The linked EVTX was parsed locally for Sysmon process-creation records. The results below are record counts from focused cross-checks, not unique action counts or Kibana result totals.

| PDF question | Evidence checked in the EVTX | Matching records |
| --- | --- | --- |
| 1 | Daily scheduled-task invocation with a credential argument | 2 |
| 2 | SDelete and C:\some\file.txt | 1 |
| 3 | SMB share access invocation mentioning Administrator | 3 |
| 4 | Ping command lines containing the 192.168.1 address prefix | 255 |
| 5 | BITSAdmin and C:\Windows\Temp\bitsadmin_flag.ps1 | 6 |

The PDF's Q3 search appears in the screenshot on page 5. Password values, full command lines, and the binary log are excluded from this repository.

[Original EVTX sample](https://github.com/sbousseaden/EVTX-ATTACK-SAMPLES/blob/master/AutomatedTestingTools/PanacheSysmon_vs_AtomicRedTeam01.evtx)
