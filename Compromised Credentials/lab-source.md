Step 1: Open the lab link to access the Windows machine.

Windows Machine

Content Image

Step 2: Check how many users are present on the machine.

Command

net user
Content Image

There are total five users are present on the Windows machine, including the default users.

Step 3: Open the Windows Event Viewer and identify any instances of multiple invalid login attempts targeting a specific user.

Select Windows Logs --> Security

Content Image

Focusing on event ID 4625 - 4625: An account failed to log on

If a user enters an incorrect username or password, the account is unable to log in. In this scenario, the computer records the log event ID 4625 when the user attempts to sign in.

Apply filter on the log event 4625

Content Image

Content Image

There are a total of 3651 events present with ID 4625, indicating a clear case of a brute force attack.

You can utilize the following PowerShell commands to count the number of failed logon attempts on the machine.

Another Way to find failed logon attempts:

# Define the log name and event ID
$logName = 'Security'
$eventID = 4625

# Get events from the Security log with Event ID 4625
$events = Get-WinEvent -LogName $logName -FilterXPath "*[System[EventID=$eventID]]"

# Count the number of events
$eventCount = $events.Count

# Print the result
Write-Host "Number of events with Event ID $eventID in $logName log: $eventCount"
Content Image

Investigate how many windows users are targeted in this brute force attack

# Define the log name and event ID
$logName = 'Security'
$eventID = 4625

# Get events from the Security log with Event ID 4625
$events = Get-WinEvent -LogName $logName -FilterXPath "*[System[EventID=$eventID]]"

# Extract unique Account Names
$accountNames = $events | ForEach-Object {
    [xml]$eventXml = $_.ToXml()
    $eventXml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
} | Get-Unique

# Print the unique Account Names
Write-Host "Unique Account Names from events with Event ID $eventID in $logName log:"
$accountNames
Content Image

Multiple users have been targeted, with only one being a valid user (Administrator), while the rest are not valid. This pattern strongly suggests that an attacker has executed a username and password spraying attack on the target machine.

To further confirm, let's identify the IP address associated with the attacker's machine. In Event ID 4625, concentrate on the Network Information field and examine the Source Network Address.

Content Image

The attacker machine IP address is: 13.214.192.125

You can run the following PowerShell script to confirm that all 4625 Event IDs are generated from this specific IP address.

Another Approach:

# Define the log name and event ID
$logName = 'Security'
$eventID = 4625

# Get events from the Security log with Event ID 4625
$events = Get-WinEvent -LogName $logName -FilterXPath "*[System[EventID=$eventID]]"

# Extract unique Source Network Addresses
$sourceNetworkAddresses = $events | ForEach-Object {
    [xml]$eventXml = $_.ToXml()
    $eventXml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select-Object -ExpandProperty '#text'
} | Get-Unique

# Print the unique Source Network Addresses
Write-Host "Unique Source Network Addresses from events with Event ID $eventID in $logName log:"
$sourceNetworkAddresses
Content Image

All login failed events with Event ID 4625 are associated with the IP address 13.214.192.125.

Step 4: With confirmation that the machine has been targeted by the attacker (13.214.192.125), further investigation is necessary to validate whether the administrator account has been compromised or not.

Concentrate on Event ID 4624, which is generated when an account successfully logs in. Analyze the details within these events to gain insights into any legitimate access to the system. Pay close attention to the associated account, timestamps, and source IP addresses in Event ID 4624 to ensure that no unauthorized access or compromises have occurred.

Apply filter on the log event 4624

Content Image

By comparing this information with the previously identified attacker's IP address, we can further validate the integrity of the administrator account and identify any potential signs of compromise or unusual activity.

Content Image

Event 4624 is recorded for the Administrator user, and the login information's source IP address indeed matches the attacker's IP address. This indicates that the administrator account has been compromised, and the attacker has successfully obtained the valid password for the account.

Investigate the sysmon log and identify the Network connection by filtering event ID by 3

Content Image

Content Image

The attacker is focusing on the SMB service to identify valid usernames and passwords. Security Event ID 4624 and Sysmon Event ID 3 both indicate a match in the source port, suggesting that the attacker has successfully obtained administrator account credentials through the SMB service.

Conclusion

The investigation in the lab environment revealed a targeted attack on a Windows machine, with an emphasis on compromising user accounts through a brute force password spraying attack. The security team successfully identified the attacker's IP address (13.214.192.125) and confirmed a breach of the Administrator account. The analysis of Event IDs 4625 and 4624 provided crucial insights, pointing towards a systematic attempt to gain unauthorized access to the SMB service.