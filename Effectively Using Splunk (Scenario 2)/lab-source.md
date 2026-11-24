SOLUTIONS
Below, you can find solutions for every task of this lab. Remember though, that you can follow your own strategy (which may be different from the one explained in the following lab.

Kali Machine

Content Image

Task 1: Try to identify a successful exploitation attempt without consulting with the provided TTPs
Once you are logged into Splunk's web management interface, click the Search & Reporting application that resides on the Apps column on your left. You should see something similar to the below.

Content Image

Before starting your investigation change the time range picker to All time.

Always identify the available sourcetypes before you begin your investigation. You can do that as follows.

| metadata type=sourcetypes index="botsv1"
You should see the below.

Content Image

As you can see, Splunk has ingested Windows event logs, Sysmon logs, Fortigate UTM logs, Suricata logs etc.

If you want better granularity regarding the available sourcetypes, submit the search below.

| metadata type=sources index="botsv1"
Notice that searches leveraging metadata are executed almost instantaneously.

You should now see something similar to the below.

Content Image

The results between the last two searches are the same. The second search will provide you with a little more detail about the available sourcetypes.

If you look carefully enough you will notice that the firstTime, lastTime and recentTime entries follow the epoch time representation. To convert epoch time to a human understandable representation submit the following search.

| metadata type=sources index="botsv1" | convert ctime(firstTime) as firstTime | convert ctime(lastTime) as lastTime | convert ctime(recentTime) as recentTime
You should see the following.

Content Image

In case you want to identify all the available hosts in the dataset before you start your investigation, you can do that through the following search.

| metadata type=hosts index="botsv1" | convert ctime(firstTime) as firstTime | convert ctime(lastTime) as lastTime | convert ctime(recentTime) as recentTime
You can sort the above by total count to gain a better understanding.
Content Image

A great sourcetype to start with is stream:dns.

index=botsv1 sourcetype=stream:dns | fieldsummary
Content Image

The results of the search above may be difficult to read, so create a table that will contain field and values entries only. You can that by submitting the following search.

index=botsv1 sourcetype=stream:dns | fieldsummary | table field values
You should see something similar to the below.

Content Image

You now need to determine which of the available fields is more important. dest could provide you with useful information, but the most interesting field in these results is query{}, since it can provide you with information related to interactions with remote (and possibly malicious) servers.

Content Image

To better analyze DNS query information, submit the following search.

index=botsv1 sourcetype=stream:dns | stats count by query{} | sort count
You should see something similar to the below.
Content Image

Going through all queries you will spot some curious-looking domain names. Such a domain name is cerberhhyed5frqa.xmfir0.win

You can look into this curious-looking domain, as follows.

index=botsv1 sourcetype=stream:dns query{}=cerberhhyed5frqa.xmfir0.win | table _time src_ip dest_ip query{}
You will see the following results.

Content Image

In the results (sorted by time) above you can see the 192.168.250.100 host making a DNS query to 192.168.250.20. 192.168.250.20 in turn makes a number of external DNS queries. From this behavior you can assume that 192.168.250.20 is a DNS server and 192.168.250.100 is probably a compromised machine.

Based on the time included in the results above, you can give 192.168.250.100 a look as follows.

index=botsv1 sourcetype=stream:dns src_ip=192.168.250.100 earliest=08/24/2016:0:0:0 | table _time src_ip dest_ip query{} | dedup query{}
You should see the following.

Content Image

Notice that the earliest events are located at the bottom of the table.

The possibly compromised 192.168.250.100 system is looking for isatap and wpad right after visiting the curious-looking cerberhhyed5frqa.xmfir0.win domain. isatap is related to IPv6 tunneling and wpad to proxying. This is quite suspicious...

What you should do next is investigate the behavior of the possibly compromised 192.168.250.100 system, by analyzing other logs for approximately the same period of time as above. Sysmon logs are perfect for this.

First, change the time range picker as follows and click Apply.

Content Image

Then, submit the following search.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" SourceIp="192.168.250.100"
You should see something similar to the below.
Content Image

Those two spikes are certainly suspicious.

An important field to check is app.

Content Image

The fact that osk.exe is in around 83% of the values is certainly suspicious

osk.exe is usually related to on screen keyboard. The known osk.exe though doesn't reside in the C:\\Users\\\\AppData\\Roaming\\ directory. This is also suspicious.

Notice a user named Bob Smith, who is a possible attack victim

You could have also identified this application, as follows.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" SourceIp="192.168.250.100" | stats count by app
Content Image

osk.exe definitely looks suspicious. So, give it a closer look by simply clicking on it.

You should see something similar to the below.

Content Image

By inspecting the dest_port field. You will be presented with the below.

Content Image

That's an awful lot of network traffic for an application like on screen keyboard. This is suspicious.

Port 6892 corresponds to bit torrent and windows live messenger file transfer, something also suspicious.

There is only one communication on port 80, click on it to learn more. You should see the below.

Content Image

There's a destination IP in the result 54.148.194.58, which is worth checking, but since user Bob Smith is most probably a victim of an attack, consult with the available Sysmon logs to identify what else is running on his machine. You can do that as follows.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" app="C:\\Users\\bob.smith.WAYNECORPINC*"
Inspect the app field once again. You should see the following.
Content Image

Notice the existence of another curious looking application C:\Users\bob.smith.WAYNECORPINC\AppData\Roaming\121214.tmp. Give it a look by clicking on it. You should see the following.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" app="C:\\Users\\bob.smith.WAYNECORPINC*" app="C:\\Users\\bob.smith.WAYNECORPINC\\AppData\\Roaming\\121214.tmp"
Content Image

Nothing curious-looking in the results, but there are important fields that could be added to assist your investigation, such as the CommandLine or the ParentCommandLine one.

Submit the following search to see all the occurrences of 121214.tmp in the Sysmon logs and also any entry/log that contains ParentCommandLine or CommandLine entries.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" "121214.tmp" AND (ParentCommandLine=* OR CommandLine=*) | table _time process process_id ParentProcessId ParentImage CommandLine ParentCommandLine
You should see the following.
Content Image

The earliest events are at the bottom of the table. If you start from the first (earliest) event you will see that wscript.exe (parent) called cmd.exe (child). In addition to that, you can see from ParentCommandLine that wscript.exe executed 20429.vbs.

You can identify more about 20429.vbs by submitting the following search.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" "20429.vbs" AND (ParentCommandLine=* OR CommandLine=*) | table _time process process_id ParentProcessId ParentImage CommandLine ParentCommandLine
You should see the following.

Content Image

This is clearly obfuscated code. User Bob Smith is definitely victim of an attack.

Sysmon logs also contain MD5 hashes. If you would like to learn more about that 121214.tmp file you saw earlier, change time range picker to All time, submit the following search and inspect the md5 field.

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" "121214.tmp" 
Content Image

You will come across the following.

Content Image

If you submit the EE0828A4E4C195D97313BFC7D4B531F1 hash on a search engine, you will identify that you are dealing with Cerber ransomware.

Task 2: Translate the provided red team TTPs into Splunk searches
Identify a malicious USB
Removable media can be identified through the following Splunk searches.

index=botsv1 sourcetype="xmlwineventlog:microsoft-windows-sysmon/operational" "d:\\" | stats count by Computer,CommandLine
Content Image

You will have to include all possible drive letters. The search above is to test the existence of a D: drive only.

index=botsv1 sourcetype=winregistry friendlyname | table host object data
Content Image

Identify computer-generated domain names
The following search may uncover computer-generated domain names.

index=botsv1 sourcetype=stream:dns | table query{} | lookup ut_parse_extended_lookup url as query{} | search ut_domain!=None NOT (ut_domain_without_tld=microsoft OR ut_domain_without_tld=msn OR ut_domain_without_tld=akamaiedge OR ut_domain_without_tld=akadns OR ut_domain=nsatc.net OR ut_domain=quest.net OR ut_domain=windows.com OR ut_domain=arin.net) | `ut_shannon(ut_subdomain)` | stats count by query{} ut_subdomain ut_domain ut_domain_without_tld ut_tld ut_shannon | sort - ut_shannon
Content Image

Identify malicious VBS
The following search may identify malicious VBS files

index=botsv1 source="WinEventLog:Microsoft-Windows-Sysmon/Operational" "*.vbs" AND (ParentCommandLine=* OR CommandLine=*) | table _time process process_id Parent rocessId ParentImage CommandLine ParentCommandLine
Content Image

Identify mature ransomware activity
The following search can possibly identify mature ransomware activity.

index="botsv1" source="wineventlog:microsoft-windows-sysmon/operational" EventCode=1 process=*\\vssadmin.exe | search CommandLine="*vssadmin*" CommandLine="*Delete *" CommandLine="*Shadows*"
Content Image

Identify code obfuscation
The following search can possibly identify attackers using code obfuscation.

index="botsv1" source="wineventlog:microsoft-windows-sysmon/operational" | eval len=len(CommandLine) | table User, len, CommandLine | sort - len
Content Image
