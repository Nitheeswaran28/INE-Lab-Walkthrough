Step 1: Open the lab link to access the Ubuntu machine.


Step 2: Visit the Wazuh dashboard and check the events/alerts.

Open the Firefox web browser and access the Wazuh Dashboard on [https://wazuh.ine.local.] Use the stored credentials for the login.


View the "Agents Summary" and click on disconnected.


Click on the agent with ID 002.


Click on "More." Then go to the "MITRE ATT&CK" dashboard.


Go to the "Framework" and select the date timeline as "24th December 2025" and hide the techniques with no alerts.


First, we can notice that there's an alert for "Create Account." Open it.


Note the timestamp and check the description "New user added to the system." The new user is "eviluser," and the agent name is "ip-10-0-0-100."


Next, open the alert for "Remote Services."


Notice here the timestamp is after the "eviluser" account creation, and the SSH service was used by the "eviluser" from the machine IP 10.0.0.11.


Next, open the alert for "Sudo and Sudo Caching."


Note the timestamp. Next, we can see that the "/etc/sudoers" file has been accessed.


We can see it was accessed by the "evil user," and a full log can also be seen.


Our initial investigation concludes that there was a suspicious new user (eviluser) creation on the agent "ip-10-0-0-100." Followed by the remote service access (SSH) and possible privilege escalation attempt. Now, let's document our findings in TheHive.

Step 3: Log in to TheHive and document the findings.

Navigate to the 2nd machine, open Firefox, and visit [http://localhost:9000.] Use the following analyst credentials:

Credentials:
```
Username: [redacted]
Password: [redacted]
```


Click on Create case.


Select the "Empty case."


Now enter the details given below:
- Title: Suspicious Account Creation Followed by Privileged Access Attempt
- Date: Current Date
- Severity: HIGH (New local user created, Remote login using that account, Attempt to modify /etc/sudoers)
- TLP: TLP:AMBER (Internal security incident, Sensitive system-level activity, Should not be freely shared)
- PAP: PAP:AMBER (Active investigation, Actions must follow SOC process, No unrestricted handling)
- Tags: privilege-escalation, account-creation, persistence, sudo, ssh
- Description: Wazuh detected the creation of a new local user account eviluser on host ip-10-0-0-100. Shortly after account creation, successful SSH authentication was observed for the same account originating from internal IP 10.0.0.11. Subsequent logs indicate that the newly created user executed sudo for the first time and attempted to modify the /etc/sudoers file, suggesting a potential privilege escalation attempt. The sequence of events indicates possible unauthorized persistence and privilege escalation activity. Findings have been documented and escalated for further review.


Click on the "Confirm" button.


The case has been created.


Next go to the "Observables" and click on the "+" button.


Add the first observable:
- Type: other
- Value: eviluser
- TLP: TLP:AMBER
- PAP: PAP:AMBER
- Is IOC: Yes
- Has been sighted: Yes
- Sighted at: 2025-12-24 07:41
- Tags: new-account, persistence, local-user
- Description: Newly created local user account observed prior to remote authentication and privileged command execution.


Click on "Save and add another."


Next observable:
- Type: ip
- Value: 10.0.0.11
- TLP: TLP:AMBER
- PAP: PAP:AMBER
- Has been sighted: Yes
- Sighted at: 2025-12-24 07:59
- Tags: ssh, remote-access
- Description: Source IP used for successful SSH authentication to the newly created account.


Click on "Save and add another."


Next observable:
- Type: hostname
- Value: ip-10-0-0-100
- TLP: TLP:AMBER
- PAP: PAP:AMBER
- Has been sighted: Yes
- Sighted at: 2025-12-24 08:00
- Tags: linux, privilege-escalation
- Description: Target Linux host involved in account creation, SSH access, and privileged command execution.


Click on "Save and add another."


Next observable:
- Type: filename
- Value: /etc/sudoers
- TLP: TLP:AMBER
- PAP: PAP:AMBER
- Is IOC: Yes
- Has been sighted: Yes
- Sighted at: 2025-12-24 08:00
- Tags: sudo, privilege-escalation, configuration
- Description: The /etc/sudoers file was accessed via sudo by the newly created user eviluser, indicating an attempt to modify privileged configuration for potential privilege escalation.


Click on the "Confirm" button.


Next, go to the "Tasks" and click on the "+" button.


Add the following task:
- Group: default
- Title: Validate legitimacy of privileged access attempt
- Mandatory: Yes
- Description: Validate whether the account creation and modification attempt of /etc/sudoers was authorized. Confirm if privilege escalation was successful and assess potential impact.
- Assignee: SOC 2


Click on the "Confirm" button.


Next we will go to "TTPs" and click on the "+" button.


Add the following TTPs:
- Catalog: Enterprise Attack
- Occur date: 2025-12-24 07:41
- Technique: T1136 – Create Account (New user eviluser was created.)
- Technique: T1078 – Valid Accounts (Successful SSH login using the newly created account.)
- Technique: T1021 – Remote Services (Remote SSH access from 10.0.0.11.)
- Technique: T1548.003 – Sudo and Sudo Caching (Attempt to modify /etc/sudoers using sudo.)
Click on the "Confirm" button.


Finally, we will change the assignee to "SOC 2" as shown below:




End of the Lab!
An initial triage was conducted on a sequence of suspicious alerts related to account creation and potential privilege escalation identified in the Wazuh dashboard. Relevant system and authentication logs were analyzed to establish context and assess potential risk. All identified observables and investigation findings have been documented in TheHive, and the case has been escalated to SOC Level 2 for further analysis and validation.