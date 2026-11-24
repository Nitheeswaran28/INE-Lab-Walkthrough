SOLUTIONS
Below, you can find solutions for every task of this lab. Remember though, that you can follow your own strategy, which may be different from the one explained in the following lab.

Kali Machine

Content Image

Task 1: Identify any reconnaissance activities against your network through Splunk searches
Once you are logged into Splunk's web management interface, click the Search & Reporting application that resides on the Apps column on your left. You should see something similar to the below.

Content Image

In order to test if Splunk can successfully access the ingested/loaded data, first change the time range picker to All time and then, submit the following search.

index="botsv1" earliest=0
You should see the number of events growing as time progresses.

Content Image

Now that we know everything worked as expected, let's identify any reconnaissance activities against Wayne Enterprises. As a reminder, the organization's website is imreallynotbatman.com.

The first thing we should do is determine the sourcetypes to search. Specifically, we should first determine the sourcetypes that are associated with imreallynotbatman.com. We can do so by changing the time range picker to All time and submitting the following Splunk search.

index=botsv1 imreallynotbatman.com
You should see something similar to the below.

Content Image

To determine the sourcetypes, simply click on sourcetype (red rectangle above). You should see the following.

Content Image

Let's also identify all source addresses. We can do so through the previous search, but this time we will scroll down and click on the src field, as follows.

Content Image

Since we are interested in identifying reconnaissance activities, it would be better to focus on the stream:http sourcetype. (Stream is a free app for Splunk that collects wire data and can focus on a number of different protocols including smtp, tcp, ip, http and so on.)

index=botsv1 imreallynotbatman.com sourcetype=stream:http
Content Image

If we do so, the sources will be narrowed down to two, 40.80.148.42 and 23.22.63.114. 40.80.148.42 is associated with ~95% of the http traffic, so let's focus on this one for the time being.

An alternative way to identify all sources is the following.

index=botsv1 imreallynotbatman.com sourcetype=stream* | stats count(src_ip) as Requests by src_ip | sort - Requests
Content Image

So far, we can only assume that 40.80.148.42 was the IP from where the APT group performed its reconnaissance/scanning activities. We can validate this finding, by checking with Suricata, as follows.

index=botsv1 imreallynotbatman.com src_ip=40.80.148.42 sourcetype=suricata
We see Suricata logs related to 40.80.148.42, but no signature field. We can see the signatures by scrolling down, clicking on more fields and choosing signature. If we do so, the signature field will be visible under the SELECTED FIELDS column.

Content Image

From the Suricata signatures that were triggered, we can conclude that 40.80.148.42 was actually scanning imreallynotbatman.com.

We are also interested in knowing our adversary's level of sophistication. So the question that arises is, did the APT group use known or sophisticated scanning techniques? Let's take a look at the submitted requests to answer that.

index=botsv1 src_ip=40.80.148.42 sourcetype=stream:http
The detailed request information can be found inside the src_headers field.

Content Image

The APT group utilized an instance of the reputable Acunetix vulnerability scanner.

We could have also identified the usage of this tool by looking for uncommon user agents.

Content Image

We can easily identify which server was the target through the same search and the dest field.

Content Image

The target was obviously 192.168.250.70.

If we want to have a closer look at what has been requested by the APT group, we can do that as follows.

index=botsv1 dest_ip=192.168.250.70 sourcetype=stream:http
The URLs being requested can be found inside the uri field.

Content Image

We are also interested in successful page loads. We can identify them, as follows.

index=botsv1 dest_ip=192.168.250.70 sourcetype=stream:http| timechart count by uri limit=10
Content Image

We could have achieved similar results through the iis sourcetype, as follows. (This time we are using a transformational search command called stats that will allow us to count the number of events grouped by URI.)

index=botsv1 sourcetype=iis sc_status=200 | stats values(cs_uri_stem)
Content Image

You may be wondering why we aren't specifying 192.168.250.70. This is because if we submit the below and check the host field, we will find only one host, we1149srv. This host's IP address is 192.168.250.70

index=botsv1 sourcetype=iis
Content Image

Below are our findings so far.

Content Image

Content Image

Task 2: Identify any weaponization activities on your network
At this point, we need to understand that Splunk is not panacea. During our investigations, not every answer can be found within the SIEM. There will be times when we will need to pivot from the SIEM to other internal or open sources to find answers.

We are interested in identifying domains that are pre-staged to attack Wayne Enterprises.

We gave the 40.80.148.42 IP address a good look through Splunk. Let's do the same for 23.22.63.114 but through open sources since Splunk doesn't contain too much information about it.

If we go to an open source like http://www.robtex.com and submit the 23.22.63.114 IP, we will come across the following.

Content Image

As we can see, this IP has a number of other domain names associated with it. These domain names are most probably phishing domains since their name is similar to the organization we work for, Wayne Enterprises.

Open sources like https://threatcrowd.org and can provide us with additional information.

Content Image

For example, through threatcrowd.org, we identified additional domains associated with the APT group we are dealing with by simply submitting the 23.22.63.114 IP.

Remember when we talked about whois information and how attackers leverage them for targeted attacks? Well, let's give attackers a taste of their own poison, by checking the whois information of every associated domain.

While checking the whois information of wayncorpinc.com we come across the following.

Content Image

We can then proceed to reverse email searches and possibly identify additional infrastructure associated with the APT group. Find an example of a reverse email search below.

https://www.threatcrowd.org/email.php?email=LILLIAN.ROSE@PO1S0N1VY.COM

Here are our findings so far:

Content Image

Task 3: Identify any delivery activities on your network
We need to know as much as possible about this APT group's TTPs and used malware, so let's dig deeper through open sources.

https://www.threatminer.org has a great capability of including related malware samples when searching for information about an IP address. This is what we will come across while searching for information about 23.22.63.114 on threatminer.org.

Content Image

We can then submit these MD5 hashes to open sources like threatminer, VirusTotal or hybrid-analysis.com to identify additional metadata about the sample(s).

Content Image

The APT group may create mutants, so hashes may not prove useful. We should note down that filename though in case they keep that.

Below are our findings so far.

Content Image

Task 4: Identify any exploitation activities on your network through Splunk searches
It is about time we go back to Splunk to identify any exploitation activities. Let's start by identifying source IP addresses that are associated with the largest number of http events. We can do that, as follows.

index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST
The src field contains what we are looking for. We specified that we are interested in POST requests since logins are usually performed through POST requests (more on that in a bit).

Content Image

Let's take a look at those POST requests made by 40.80.148.42

index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST src_ip="40.80.148.42"
The form_data field contains information that we want to check when dealing with POST requests.

Content Image

Nothing to justify successful exploitation activities. Let's check 23.22.63.114.

index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST src_ip="23.22.63.114"
Content Image

It looks like 23.22.63.114 is brute forcing the web server's authentication.

Let's make sure, as follows.

index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" http_method=POST form_data=*username*passwd* | stats count by src_ip
Content Image

Indeed 23.22.63.114 performed a brute force attack against the web server's authentication.

We are quite interested in knowing if the brute force attack was successful. We can determine that, as follows.

[Submitted-password extraction search redacted.]
Content Image
[Description of password extraction redacted. Repeated requests do not prove successful authentication.]

If we want to get an idea of the time of the compromise and the URI that was targeted, we can do that as follows.

[Submitted-password extraction search redacted.]
Content Image

Finally, if we want to view the two successful logins we can do so, as follows.

[Submitted-password extraction search redacted.]
Content Image

Below are our findings so far.

Content Image

Content Image

Task 5: Identify any installation activities on your network through Splunk searches
As far as the installation phase of the cyber kill chain is concerned, we are mostly interested in identifying any malware being uploaded.

We can identify that through various sourcetypes, specifically, stream:http and Suricata.

index=botsv1 sourcetype=stream:http dest_ip="192.168.250.70" *.exe
The part_filename{} field contains the information we want to check. It won't be visible by default, so add it.
Content Image

Content Image

index=botsv1 sourcetype=suricata (dest=imreallynotbatman.com OR dest="192.168.250.70") http.http_method=POST .exe
The fileinfo.filename field contains the information we want to check.

Content Image

3791.exe must be the uploaded malware.

The source from where the file was uploaded can easily be identified, as follows.

index=botsv1 sourcetype=suricata dest_ip="192.168.250.70" http.http_method=POST .exe
Content Image

It would be great if we could also identify the hash of the uploaded file. But what sourcetype should we use? Let's find out, as follows.

index=botsv1 3791.exe
Content Image

Sysmon is a good candidate since it logs information such as MD5, SHA1 and SHA256 hashes of files.

index=botsv1 3791.exe sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
Content Image

The important fields, in this case, are Hashes, CommandLine and ParentCommandLine. You will have to add the last two ones since they are not visible by default.

Before analyzing the results, let's narrow things down a little bit.

index=botsv1 3791.exe sourcetype=XmlWinEventLog:Microsoft-Windows-Sysmon/Operational EventCode=1
Content Image

The search above includes EventCode 1 since this EventCode is related to process creation events. Unfortunately, to get the MD5 hash of the uploaded file, we need to narrow things down even further. Specifically, we will need to search for 3791.exe inside the command line field, since this field captures the process starting.

So, the final search is the following.

index=botsv1 3791.exe CommandLine=3791.exe | stats values(md5)
Content Image

Below are our findings so far.

Content Image

Content Image

Task 6: Identify any command and control-related activities on your network through Splunk searches
As far as the Command and Control phase of the cyber kill chain is concerned, we are mostly interested in identifying any domain used for command and control purposes.

We have a powerful ally inside Splunk to assist us in answering such questions. This ally is the stream.dns sourcetype. Since we already know that 23.22.63.114 is of concern, we can utilize Splunk and the stream.dns sourcetype to identify DNS events where 22.23.63.114 was the answer.

index=botsv1 sourcetype=stream:dns 23.22.63.114 | stats values("name{}")
Content Image

If you look carefully enough, you will identify that the prankglassinebracket.jumpingcrab.com domain has been used by attackers to deface the web server. Give it a try...

References:

https://www.slideshare.net/Splunk/splunk-data-onboarding-overview-splunk-data-collection-architecture

https://www.splunk.com/en_us/training.html