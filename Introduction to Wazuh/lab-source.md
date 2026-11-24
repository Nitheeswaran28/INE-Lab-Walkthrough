# Introduction to Wazuh — sanitized source notes

This transcription summarizes the pasted solution and attached lab overview. It retains the agent setup commands and reported findings. Dashboard credentials, operational password-guessing commands, the certificate-warning instruction, and external solution-image links were omitted.

## Step 1 — Access the SOC machine

Open the Wazuh Dashboard at `https://localhost` on the SOC machine. The initial dashboard shows no connected agents.

## Step 2 — Configure the Linux agent

Run `ip addr` on the SOC machine. The example SOC IP is `10.0.18.138`.

On Target Ubuntu:

```bash
sed -i 's/MANAGER_IP/10.0.18.138/g' /var/ossec/etc/ossec.conf
cat /var/ossec/etc/ossec.conf
/etc/init.d/wazuh-agent start
/etc/init.d/wazuh-agent status
```

The example Ubuntu address is `10.0.31.185`. Refresh the dashboard to see one active agent. Open the agent and go to **More > MITRE ATT&CK > Framework**. Check the timeline and toggle **Hide techniques with no alerts**.

## Step 3 — Review Linux authentication alerts

The exercise shows simulated SSH password guessing and reports a successful attempt. Refresh the dashboard, open **Brute Force**, and expand **Multiple authentication failures followed by a success**. The expanded alert contains the full log and source and target addresses.

## Step 4 — Configure the Windows agent

The lab MSI is on the Target Windows Desktop. Use the current SOC machine IP for the manager and registration server.

```powershell
cd Desktop
msiexec.exe /i wazuh-agent.msi /q WAZUH_MANAGER=10.0.18.138 WAZUH_REGISTRATION_SERVER=10.0.18.138 WAZUH_AGENT_GROUP=default
Start-Service -Name WazuhSvc
ipconfig
```

The notes say the RDP session might disconnect after the service starts. Reconnect if needed. The example Windows address is `10.0.20.137`. Refresh Wazuh and confirm two connected agents. Open the Windows agent’s **More > MITRE ATT&CK > Events** view.

## Step 5 — Review Windows authentication events

The exercise shows simulated RDP password guessing and reports a successful attempt. The solution says new events appeared in the Windows agent’s **Events** tab.

## Conclusion

Both agents connected to the Wazuh manager. The dashboard showed alerts or events associated with the authentication scenarios.
