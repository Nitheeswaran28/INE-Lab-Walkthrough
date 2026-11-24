The organization you work for (Wayne Enterprises) is using [Splunk](https://www.splunk.com/) as a SIEM solution to enhance its intrusion detection capabilities. The SOC manager informed you that the organization has been hit by an APT group. He tasked you with responding to this incident by heavily utilizing Splunk and all the data that it ingested.

The data that Splunk has ingested consist of Windows event logs, Sysmon logs, Fortinet next-generation firewall logs, Suricata logs, etc.

Note: This lab is based on the [Boss Of The SOC (BOTS) v1 dataset](https://github.com/splunk/botsv1) released by Splunk. Credits to [Ryan Kovar](https://twitter.com/meansec), [Dave Herrald](https://twitter.com/daveherrald?lang=el) and [John Stoner](https://twitter.com/stonerpsu) for sharing the Splunk detection tips this lab covers with the public, through this dataset.
The learning objective of this lab is to not only get familiar with Splunk's architecture and detection capabilities but also to learn effective Splunk search writing.

Specifically, you will learn how to use Splunk's capabilities in order to:
- Have better visibility over a network
- Respond to incidents timely and effectively
- Proactively hunt for threats
Splunk's creators describe it as a solution to aggregate, analyze and get answers from machine data. Splunk can be used for Application Management, Operations Management, Security & Compliance, etc.

When it comes to security, Splunk can be used as a log management solution but most importantly as an analytics-driven SIEM. Splunk can fortify investigations of dynamic, multi-step attacks with detailed visualizations and even enhance an organization's detection capabilities through User Behavior Analytics.

Splunk can literally ingest almost any data from almost any source, through both an agent-less and a forwarder approach.

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/0.png)

Splunk Architecture Overview:

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/1.png)

Splunk's architecture (at a high level) consists of the:
- Forwarder component
- Universal Forwarders collect data from remote sources and send them to one or more Splunk Indexers. Universal Forwarders are separate downloads that can be installed on any remote source, with little impact on network or host performance.
  - Heavy Forwarders also collect data from remote sources, but they are typically used for heavy data aggregation tasks, from sources like firewalls or data routing/filtering passing points. According to [Splexicon](https://docs.splunk.com/Splexicon:Heavyforwarder), unlike other forwarder types, heavy forwarders parse data before forwarding them and can route data based on criteria such as source or type of event. They can also index data locally while forwarding the data to another indexer. Heavy Forwarders are usually run as "data collection nodes" for API/scripted data access, and they are only compatible with Splunk Enterprise.
Note: HTTP Event Collectors (HECs) also exist to collect data directly from applications, at-scale, through a token-based JSON or raw API way. Data are sent directly to the Indexer level.
- Indexer component
The Indexer processes machine data, storing the results in indexes as events, enabling fast search and analysis. As the indexer indexes data, it creates a number of files organized in sets of directories by age. Each directory contains raw data (compressed) and indexes (points to the raw data).
- Search Head component
The Search Head component allows users to use the Search language to search for indexed data. It distributes user search requests to the Indexers and consolidates the results as well as extracts field value pairs from the events to the user. Knowledge Objects on the Search Heads can be created to extract additional fields and transform the data without changing the underlying index data. It should be noted that Search Heads also provide tools to enhance the search experience such as reports, dashboards, and visualizations.

Splunk Apps and Technology Add-ons (TAs):

Splunk Apps are designed to address a wide variety of use cases and to extend the power of Splunk. Essentially, they are collections of files containing data inputs, UI elements, and/or knowledge objects. Splunk Apps also allow multiple workspaces for different use cases/user roles to co-exist on a single Splunk instance. Ready-made apps are available on Splunkbase (splunkbase.com).

Splunk Technology Add-ons abstract the collection methodology and they typically include relevant field extractions (schema-on-the-fly). They also include relevant config files (props/transforms) and ancillary scripts binaries.

You can think of a Splunk App as a complete solution, that typically uses one or more Technology Add-ons.

Splunk Users and Roles:

Splunk users are assigned roles which determine their capabilities and data access. Out of the box, there are three main roles:
- admin: This role has the most capabilities assigned to it.
- power: This role can edit all shared objects (saved searches, etc.) and alerts, tag events, and other similar tasks.
- user: This role can create and edit its own saved searches, run searches, edit its own preferences, create and edit event types, and other similar tasks.
Splunk's Search & Reporting App:

You will spend most of your time inside Splunk's Search & Reporting.

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/2.jpg)


[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/3.jpg)

Data Summary can provide you with hosts, sources or sourcetypes on separate tabs.

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/4.jpg)


[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/4_1.jpg)


[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/4_2.jpg)


[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/4_3.jpg)

Finally, this is how Events will look like.

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/5.jpg)

Splunk's Search Processing Language (SPL):

According to Splunk, SPL combines the best capabilities of SQL with the Unix pipeline syntax allowing you to:
- Access all data in its original format
- Optimize for time-series events
- Use the same language for visualizations
SPL provides over 140 commands that allow you to search, correlate, analyze and visualize any data.

The below diagram represents a search, broken down to its syntax components.

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/6.png)

Searches are made up of five basic components.
1. Search terms, where you specify what you are looking for. Search terms contain keywords, phrases, Booleans, etc.
2. Commands, where you specify how you want to manipulate the results. For example, create a chart, compute statistics, etc.
3. Functions, where you specify how exactly do you want to chart, compute or evaluate the results.
4. Arguments, in case there are variables you want to apply to a function.
5. Clauses, where you specify how exactly do you want to group or rename the fields in the results.
As you write searches, you will notice that some parts of the search string are automatically colored. The color is based on the search syntax. Example:

[Content Image](https://assets.ine.com/content/ptp/Effectively_Using_Splunk_Scenario_1/7.png)

Something else to consider while submitting searches is Splunk's search modes.

There are three search modes:
- Fast Mode: Field discovery off for event searches. No event or field data for stats searches.
- Smart Mode: Field discovery on for event searches. No event or field data for stats searches.
- Verbose Mode: All event & field data.
It is recommended to start searching with Smart and then go from there.

We strongly suggest you spend time studying the] [Exploring Splunk](https://www.splunk.com/goto/book) [e-book before proceeding to the lab's tasks. Especially Chapter 4, as that covers the most commonly used search commands.

Various Search aspects are also nicely documented, in the following resource.

[https://docs.splunk.com/Documentation/Splunk/7.2.4/Search/GetstartedwithSearch](https://docs.splunk.com/Documentation/Splunk/7.2.4/Search/GetstartedwithSearch)
- Splunk
- Splunk
- Use Firefox browser to connect to Splunk's web interface (http://demo.ine.local:8000)
Throughout this lab, we will split attacker actions based on the [Cyber Kill Chain](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html).