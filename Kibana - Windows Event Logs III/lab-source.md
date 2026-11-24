# Kibana: Windows Event Logs III source notes

INE/AttackDefense walkthrough 1186 supplies the three questions and Kibana filters. The matching upstream EVTX was parsed locally for Sysmon Event ID 1. One PowerShell process had an IIS parent with DefaultAppPool in its arguments. Static Base64 decoding as UTF-16LE confirmed the key in the PDF. The PowerShell process GUID matched 39 direct child records, all for appcmd.exe.

The repository stores the findings and overview image. The raw capture and encoded and decoded payload text are outside the walkthrough.

[Matching upstream sample](https://github.com/sbousseaden/EVTX-ATTACK-SAMPLES/blob/master/Credential%20Access/discovery_sysmon_1_iis_pwd_and_config_discovery_appcmd.evtx)
