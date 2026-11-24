# Kibana: Windows Event Logs II source notes

The INE/AttackDefense PDF, walkthrough 1185, provides the lab results. Pages 3 and 4 show the per-port records. Pages 5 through 8 document the aggregation and manual count. Pages 8 and 9 show the hostname lookup.

- The shared destination between TCP 389 and 445 is 10.59.4.11. The screenshot labels TCP 389 as LDAP.
- The seven destinations shown on TCP 445 are 10.59.4.11, 10.59.4.12, and 10.59.4.21 through 10.59.4.25.
- The DEV_SERVER event shows 10.59.4.12 on TCP 445.

The three answers were independently verified against nine Sysmon Event ID 3 records from 10.59.4.20 in [the matching upstream EVTX sample](https://github.com/sbousseaden/EVTX-ATTACK-SAMPLES/blob/master/Discovery/discovery_sysmon_3_Invoke_UserHunter_SourceMachine.evtx).

[Upstream EVTX collection](https://github.com/sbousseaden/EVTX-ATTACK-SAMPLES)
