SOLUTIONS
Kali Machine

Content Image

Below, you can find solutions for every task of this lab. Remember though, that you can follow your own strategy (which may be different from the one explained in the following lab.

Task 1: Add any fields you see fit to enhance your understanding of the data
Once you are connected to Kibana, first change the time picker to Last 15 years and then submit an empty search. You should come across the following.

Content Image

As we can see, the documents (or events if you like) table consists of two columns only. We can enhance our understanding of the gathered events by adding more fields. To do so, we simply click on Available Fields and then click the add button that appears next to each field upon mouse hover.

Content Image

A good start would be adding the event_id field and the computer_name field. Then, the results will look like the ones depicted below.

Content Image

Go through each available field and experiment with how documents/events are presented until you feel comfortable enough to start your analysis.

At any time, you can remove an added field simply by hovering over it and pressing the remove button that appears.

Content Image

Task 2: Create an actionable visualization
To identify all users included in the dataset you can start by submitting an empty search, expanding Available Fields and then inspecting the event_data.User field. If you do you so will come across the below.

Content Image

Do you notice that 500 records message? This is because, by default, results are limited to 500 records. You can change that by going to the Management tab and then clicking Advanced Settings, but let's create a visualization instead.

To do so, click on the Visualize tab and press the button with the cross.

Content Image

Then, click on Data Table.

Content Image

Now, click on the logstash-* index...

Content Image

and submit the search below.

event_data.User:*
The search above can be translated as winlog.event_data.User:"exists". This means that it will provide us with all the documents that contain the specified field. To identify all users, we need to create an aggregation. To create one, click the Split Rows tab and choose Terms from the Aggregation drop-down menu. On the Field drop-down menu choose winlog.event_data.User.keyword. Finally, click the play button on your upper right.

Content Image

You should now see the below

Content Image

These results above do not contain all users, but only five of them. Why so? This is because of the default Size of the aggregation we created being five. To see all users we can specify a larger size like fifty (50) and then press the play button once again.

Content Image

You can save this visualization if you like by pressing Save on your upper right and specifying a name.

If we would like a more actionable visualization we can focus, for example, on the Windows Security Log Event ID 4776 and the Windows Security Log Event ID 4625 events. Both of them can be used to identify unsuccessful logins.

Suppose that we want to be able to see when unsuccessful login attempts occurred.

First we have to create the appropriate ELK search.

One viable search is the below.

(event_id:4776 AND -keywords:"Audit Success") OR event_id:4625
Event ID 4625 is related solely to unsuccessful login attempts, but Event ID 4776 is related to computers attempting to validate the credentials for an account. When a successful login attempt occurs the corresponding Event ID 4776 document will contain an "Audit Success" keyword and an Error Code "0x00" (see below).
Content Image

Obviously we want those out, hence the AND -keywords:"Audit Success" part of the search.

To create such a visualization we click Visualize, we press the button with the cross and we choose Area

Content Image

Then, we click on the logstash-* index, we submit the search above and we click on X-Axis.

There, we choose Date Histogram on the Aggregation drop-down menu, @timestamp on the Field drop down menu and Monthly on the Interval drop-down menu. If you do so and press the play button, you should see the following.

Content Image

Task 3: Create a search to identify files that are named like system processes
The important fields to focus on are event_data.Image and event_data.TargetFilename.

A viable search to detect files that are named like legitimate Windows processes but are located in a path other than the expected one is the below.

( event_data.Image:("*\\rundll32.exe" "*\\svchost.exe" 
"*\\wmiprvse.exe" "*\\wmiadap.exe" "*\\smss.exe" "*\\wininit.exe" "*\\taskhost.exe" "*\\lsass.exe" "*\\winlogon.exe" "*\\csrss.exe" "*\\services.exe" "*\\svchost.exe" "*\\lsm.exe" "*\\conhost.exe" "*\\dllhost.exe" "*\\dwm.exe" "*\\spoolsv.exe" "*\\wuauclt.exe" "*\\taskhost.exe" "*\\taskhostw.exe" "*\\fontdrvhost.exe" "*\\searchindexer.exe" "*\\searchprotocolhost.exe" "*\\searchfilterhost.exe" "*\\sihost.exe") AND -event_data.Image:("*\\system32\\*" "*\\syswow64\\*" "*\\winsxs\\*") ) OR ( event_data.TargetFilename:("*\\rundll32.exe" "*\\svchost.exe" "*\\wmiprvse.exe" "*\\wmiadap.exe" "*\\smss.exe" "*\\wininit.exe" "*\\taskhost.exe" "*\\lsass.exe" "*\\winlogon.exe" "*\\csrss.exe" "*\\services.exe" "*\\svchost.exe" "*\\lsm.exe" "*\\conhost.exe" "*\\dllhost.exe" "*\\dwm.exe" "*\\spoolsv.exe" "*\\wuauclt.exe" "*\\taskhost.exe" "*\\taskhostw.exe" "*\\fontdrvhost.exe" "*\\searchindexer.exe" "*\\searchprotocolhost.exe" "*\\searchfilterhost.exe" "*\\sihost.exe") AND -event_data.TargetFilename:("*\\system32\\*" "*\\syswow64\\*" "*\\winsxs\\*") )
AND -event_data.Image is excluding the expected paths.

event_data.TargetFilename is used in case the file included in the event_data.Image field interacted with another file. For example, if PowerShell downloaded a file named 65536.exe you would see the below.

​ event_data.Image: C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe

​ event_data.TargetFilename: C:\Users\PhisedUser\AppData\Local\Temp\65536.exe

If you submit the search above, you should see 6 hits.

Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

Task 4: Create a search to identify suspicious services interacting with an executable from the Windows folder
When it comes to suspicious service detection the Windows Security Log Event ID 4697 and the Windows Event ID 7045 events will prove useful. The same applies for Autoruns logs.

A viable search to identify suspicious services interacting with an executable from the windows folder is the following.

(event_id:("4697" "7045") OR (log_name:Autoruns AND event_data.Category:Services) ) AND 
event_data.CommandLine.keyword:/.*%[s|S][y|Y][s|S][t|T][e|E][m|M][r|R][o|O][o|O][t|T]%\\[^\\]*\.exe/ AND -event_data.CommandLine:(*paexe* *psexesvc* *winexesvc* *remcomsvc*)
event_id:("4697" "7045") is used to identify services being installed.

log_name:Autoruns AND event_data.Category:Services is used to identify auto-start services detected by the Autoruns MS tool. [event_data.Category:Services is used to limit the Autorun-derived documents to those only related to services]

-event_data.CommandLine:(*paexe* *psexesvc* *winexesvc* *remcomsvc*) excludes services that interact with expected Windows executables inside the Windows folder.

If you submit the search above, you should see 7 hits.

Content Image

Task 5: Create a search to identify suspicious code injection
Sysmon contains a CreateRemoteThread event (Event ID 8) that detects when a process creates a thread in another process. Malware usually do that so that the target process can load a malicious DLL (whose path is written in the virtual address space of the target process) or a malicious portable executable.

Content Image

It should be noted that remote threads can be created on a Windows system for legitimate purposes as well. Such an example is EtwpNotificationThread, which is related to threads (thread entry points actually) being "created" in the context of a process, so that certain tasks can be performed on behalf of the kernel.

To conclude the task, a viable search to identify suspicious code injection (based on Sysmon's Event ID 8) is the following.

event_id:8 AND source_name:"Microsoft-Windows-Sysmon" AND -(event_data.SourceImage:"*\\VBoxTray.exe" AND event_data.TargetImage:"*\\csrss.exe") AND -(event_data.StartFunction:EtwpNotificationThread AND event_data.SourceImage:"*\\rundll32.exe")
If you submit the search above, you should see 53 hits.
Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

wce that you see on the event_data.SourceImage field is a known Credential Dumping tool that targets the Local Security Authority Subsystem Service (LSASS) process (whose memory contains a variety of credentials).

Task 6: Create a search to identify possible privilege escalation via weak service permissions
As mentioned in this task's description attackers will interact with the sc Windows executable in order to identify if a service has weak permissions and if they have any kind of privileged access over it.

If they have enough privileges, attackers may also attempt to specify an executable of their own to be executed by the insufficiently secure service. This can be done again through the sc executable and the config option (binPath = will go next).

We can search for possible privilege escalation via weak service permissions as follows.

event_data.Image:"*\\sc.exe" AND (event_data.CommandLine:(*start* *sdshow*) OR (event_data.CommandLine:*config* AND event_data.CommandLine:*binPath*)) AND event_data.IntegrityLevel:Medium
event_data.IntegrityLevel:Medium ensures that we don't get results from privileged users (such as admins) performing legitimate service tasks.
If you submit the search above, you should see 3 hits.

Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

Task 7: Create a search to identify possible Windows session hijacking
As already mentioned in this task's description we can focus on any tscon invocation. More specifically, we are interested in any tscon invocation with SYSTEM-level privileges.

A viable search to identify possible Windows session hijacking via the described attacker technique is the below.

event_data.Image:"*\\tscon.exe" AND event_data.User:"NT AUTHORITY\\SYSTEM"
Alternatively, you can use the following search.

event_data.Image:"*\\tscon.exe" AND (event_data.LogonId:0x3e7 OR event_data.SubjectLogonId:0x3e7 OR event_data.User:"NT AUTHORITY\\SYSTEM")
If you submit any of the searches above, you should see 1 hit.
Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

Task 8: Create a search to identify the whoami command being executed with System privileges
It is quite trivial to create a search to detect the whoami command being executed with SYSTEM-level privileges.

A viable search is the following.

event_data.Image:"*\\whoami.exe" AND (event_data.LogonId:0x3e7 OR event_data.SubjectLogonId:0x3e7 OR event_data.User:"NT AUTHORITY\\SYSTEM")
The LogonIds used in this and the previous tasks are usually met when SYSTEM-level access is involved.
If you submit the search above, you should see 4 hits.

Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

Task 9: Create a search to identify LSASS loading a library not signed by Microsoft
Sysmon's Event ID 7: Image loaded can be used to monitor the DLLs being loaded by a specific process. Thankfully this event contains information about the library's signature in its data (Signature entry).

A viable search to identify LSASS loading a library not signed by Microsoft is the following.

event_id:7 AND event_data.Image:"*\\lsass.exe" AND -event_data.Signature:*Microsoft*
If you submit the search above, you should see 54 hits.
Content Image

If you are not able to see the exact same result representation, you can add the respective fields by following the steps mentioned in task 1.

The DLL being loaded (wceaux.dll) is the one the credential dumping tool you identified in task 5 uses to gather or alter credentials.

Additional Resources:

https://speakerdeck.com/felipead/elasticsearch-workshop

https://drive.google.com/file/d/0B2S_IOa0MiOHWndxWFRiUHNoNW8/view