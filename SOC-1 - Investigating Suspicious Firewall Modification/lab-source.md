Step 1: On the Splunk tab, open Firefox and navigate to http://localhost:8000.

Content Image

Step 2: Go to Search & Reporting. Set the time frame to "All time". Note that the events to be analyzed were generated in December 2025.

Content Image

Step 3: Apply the following filter to list all events:

Filter:

index=syntrix
Content Image

We see many events, but we are interested in those related to the firewall. So, let’s query for Event ID 4947 in Security logs to identify modifications to Windows Firewall rules, if any.

Filter:

index=syntrix source="WinEventLog:Security" EventCode=4947
Content Image

Content Image

We observe three firewall rule modification events on PC02 occurring within seconds of each other. These related events are part of a single action in which built-in SMB inbound firewall rules were modified across all profiles. Specifically, the rules modified were: "File and Printer Sharing (SMB-In)" and "File Server / SMB inbound rule".

But what were these modifications, and who made them? To find out, let’s filter for Event ID 2005 in the Windows Firewall logs.

Filter:

index=syntrix source="WinEventLog:Microsoft-Windows-Windows Firewall With Advanced Security/Firewall" EventCode=2005
Content Image

We observe some events.

Content Image

These events can be summarized as follows:

At around 04:35 AM on PC02, the built-in Windows Firewall rules: "File and Printer Sharing (SMB-In)" and "File Server Remote Management (SMB-In)" were modified. The changes were made by the user identified by the SID: S-1-5-21-3250466395-192540623-711299986-1010 via WMI (WmiPrvSE.exe) to "Allow" inbound TCP traffic.

Since WMI (WmiPrvSE.exe) was used to modify the firewall rules, it’s likely that the change was executed via a script or automation, such as PowerShell.

Let's review the PowerShell logs.

Filter:

index=syntrix source="WinEventLog:Microsoft-Windows-PowerShell/Operational" EventCode=4104
Content Image

We observe a number of events. However, the following specific events reveal something interesting:

Content Image

Content Image

Summary:

PowerShell executed commands to update inbound SMB firewall rules:

"File Server Remote Management (SMB-In)" → Allow any remote address

"File and Printer Sharing (SMB-In)" → Allow any remote address

User: S-1-5-21-3250466395-192540623-711299986-1010

The SMB service exposure with “Any” remote address presents a potential security risk, as it allows inbound SMB connections from any IP. This exposes the SMB service to the public internet. Normally, Windows SMB rules are restricted to local subnets or trusted networks, so a setting of ‘Any’ is unusual and potentially unsafe.

This may explain the sudden, unexpected brute-force traffic on a service (SMB). Let’s review the SMB logs to investigate further:

Filter:

index=syntrix source="WinEventLog:Microsoft-Windows-SMBServer/Security"  EventCode=551
Event ID 551 logs record failed SMB authentication attempts.

Content Image

As expected, we observed a large number of failed SMB authentication events on PC02 from external IP addresses, suggesting a brute-force attack.

For the report, we need a few more details: the username of the account that modified the firewall rules, and the IP address of PC02, which was the target of the brute-force attack. These can be identified using the following filters:

Filter:

index=syntrix "S-1-5-21-3250466395-192540623-711299986-1010" EventCode=4103
Check the PowerShell logs to identify the user associated with the SID.

Content Image

The user identified is brandon.

Filter:

index=syntrix source="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3
Review Sysmon logs to identify relevant network connection events.

Content Image

The destination IP address, 10.0.0.101, identifies PC02 as the victim.

Step 4: Now, switch to TheHive tab. Open Firefox and navigate to http://localhost:9000. Log in using the following credentials:

Username: [redacted]

Password: [redacted]

Content Image

Step 5: Click on Create case > Empty case.

Content Image

Fill in the details.

Title: Suspicious Modification of Windows Firewall Rules Leading to SMB Brute-Force Activity

Date: Current date and time.

Severity: High

TLP: TLP:Amber (Indicates limited sharing; can be shared within your organization and with its clients on a need-to-know basis.)

PAP: PAP:AMBER (Indicates that only passive checks are allowed. You can query third-party services (WHOIS, VirusTotal, etc.). You can set up honeypots to monitor traffic, but no direct interaction with the target.)

Tags: windows-firewall, smb, brute-force, powershell

Description: Security log analysis identified a series of Windows Firewall rule modifications on host PC02. Built-in SMB-related inbound firewall rules were altered to allow inbound TCP traffic from any remote address. Shortly after the firewall changes, a significant spike in failed SMB authentication attempts was observed from multiple external IP addresses, consistent with automated brute-force activity. This suggests that the firewall misconfiguration directly exposed the SMB service to the internet, enabling the observed attack traffic.

Skip the "Tasks" for now and click Confirm. Your case will be created.

Content Image

Content Image

Step 6: Next, go to Observables and click the "+" button to add a new observable.

Content Image

Fill in the details.

1. System affected due to Firewall Rule Modification

Type: ip
Value: 10.0.0.101
TLP: TLP:Amber
PAP: PAP:Amber
Is IOC: no (Internal victim system)
Has been sighted: yes
Sighted at: 01/12/2025 (04:35:35)
Ignore Similarity: no
Tags: internal-ip, smb, firewall-exposure, victim
Description: Internal IP address associated with PC02. Firewall rules were modified to allow inbound SMB traffic from any remote address, exposing this system to external authentication attempts.
Content Image

Next, click "Save and add another".

2. PowerShell Command Used to Modify Firewall Rules

Type: other
Value: Set-NetFirewallRule (SMB inbound rules modified)
TLP: TLP:Amber
PAP: PAP:Amber
Is IOC: no (Legitimate command potentially misused)
Has been sighted: yes (observed in PowerShell Operational logs)
Sighted at: 01/12/2025 (04:35:35)
Ignore Similarity: no
Tags: powershell, living-off-the-land, firewall
Description: PowerShell commands leveraging Set-NetFirewallRule were used to alter inbound SMB firewall rule configuration. While the command itself is legitimate, its use resulted in expanded network exposure.
Content Image

Similarly, add the other observables:

3. User Account Associated with Firewall Rule Changes

Type: other
Value: brandon
TLP: TLP:Amber
PAP: PAP:Amber
Is IOC: no (Legitimate user account; potential misuse or compromise)
Has been sighted: yes (identified via PowerShell and Security logs)
Sighted at: 01/12/2025 (04:35:35)
Ignore Similarity: no
Tags: user-account, firewall-modification, powershell
Description: User account associated with PowerShell activity that modified inbound SMB firewall rules.
4. Modified SMB Firewall Rules

Type: other
Value: File and Printer Sharing (SMB-In); File Server Remote Management (SMB-In)
TLP: TLP:Amber
PAP: PAP:Amber
Is IOC: no (Configuration change increasing attack surface)
Has been sighted: yes (Event IDs 4947 and 2005)
Sighted at: 01/12/2025 (04:35:35)
Ignore Similarity: no
Tags: smb, firewall-rule, misconfiguration
Description: Built-in Windows Firewall rules were modified to allow inbound SMB connections from any remote address, significantly increasing exposure to external attacks.
5. Failed SMB Authentication Attempts

Type: other
Value: Event ID 551 (SMB authentication failures)
TLP: TLP:Amber
PAP: PAP:Amber
Is IOC: no (Attack activity indicator)
Has been sighted: yes (observed in SMB Server Security logs)
Sighted at: 01/12/2025 (04:40:35) (following firewall rule modification)
Ignore Similarity: no
Tags: smb, brute-force, authentication-failure
Description: Large volume of failed SMB authentication attempts originating from external IP addresses, consistent with automated brute-force activity against the exposed SMB service.
After all observables are added, click "Confirm".

Content Image

Step 7: Next, go to TTPs. Click the "+" button to add TTPs.

Content Image

Set the "Occur date" to: 01/12/2025 (04:35:35)
Select the following Techniques:

Defense Evasion:

T1562.004 – Impair Defenses: Disable or Modify System Firewall
Reason: Firewall rules were modified to allow inbound SMB connections from any remote address, weakening host-based network protections.
Execution:

T1059.001 – Command and Scripting Interpreter: PowerShell
Reason: PowerShell was used to modify Windows Firewall rules via Set-NetFirewallRule, indicating scripted or automated configuration changes.
Content Image

After adding the TTPs, click "Confirm"

Content Image

Step 8: Next, go to Tasks. Click the "+" button to add a task.

Content Image

Fill in the task details, and assign it to an SOC 2 analyst (soc2@syntrix.com).

Task 1 — Affected Host and User Context

Title: Context on Affected Host and Associated User Account

Description: The host identified as PC02 (IP address 10.0.0.101) was the target of the observed inbound SMB traffic. The firewall modifications were executed under the user account brandon. Reviewing recent activity on this host and account may help establish whether the actions were intentional, misconfigured, or potentially the result of account misuse.

Content Image

Next, click "Save and add another".

Task 2 — Potential Impact Assessment of Brute-force Activity

Title: Context on Potential Impact from Observed SMB Brute-force Attempts

Description: A sustained volume of failed SMB authentication attempts was observed targeting PC02 following the firewall rule changes. While the events indicate unsuccessful logon attempts, additional review may help determine whether the activity resulted in any measurable impact, such as account compromise or service disruption.

After adding all the tasks, click "Confirm".

Content Image

Finally, escalate the case to SOC level 2 by changing the case owner to an SOC 2 analyst (soc2@syntrix.com), as shown below:

Content Image

Content Image

Conclusion
The investigation of firewall and security logs revealed that inbound SMB firewall rules on PC02 were modified to allow connections from any remote address. These changes were executed via PowerShell under the user account brandon. Shortly after the rule modifications, a significant increase in failed SMB authentication attempts from external IP addresses was observed, consistent with automated brute-force activity targeting the exposed service. All relevant observables have been documented, and the case has been escalated to SOC Level 2 to further assess the intent behind the firewall changes, evaluate potential account or system impact, and determine whether additional containment or remediation actions are required.