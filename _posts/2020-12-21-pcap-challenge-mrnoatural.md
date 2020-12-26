---
title:  "Pcap Challenge: Mr. Natural"
date:   2020-12-21 11:12:46
categories: [NFAT, DFIR, packet/log analysis, challenge]
tags: [pcap, network forensics, packet/log analysis, brim]
---


For the past few month's, [Internet Storm Center (ISC)](https://isc.sans.edu) handler Brad Duncan from [malware-traffic-analysis.net](http://www.malware-traffic-analysis.net) puts out a PCAP challenge (quiz) for the community to solve. The latest challenge is called [Mr. Natural](https://isc.sans.edu/forums/diary/Traffic+Analysis+Quiz+Mr+Natural/26844/). Brad provides a break down of the environment. 

- LAN segment range: 10.12.1.0/24 (10.12.1.0 thru 10.12.1.255)
- Domain: mrnatural.info
- Domain controller: 10.12.1.2 - MrNatural-DC
- LAN segment gateway: 10.12.1.1
- LAN segment broadcast address: 10.12.1.255

To successfully complete this challenge we must answer the following questions:

- What is the IP address of the infected Windows host?
- What is the MAC address of the infected Windows host?
- What is the host name of the infected Windows host?
- What is the Windows user account name used on the the infected Windows host?
- What is the date and time of this infection?
- What is the SHA256 hash of the EXE or DLL that was downloaded from 5.44.43.72?
- Which two IP addresses and associated domains have HTTPS traffic with "Internet Widgets Pty" as part of the certificate data?
- Based on the alert for CnC (command and control) traffic, what type of malware caused this infection?

To navigate this challenge, I chose to go with Brim Security's [brim](https://github.com/brimsec/brim). Before diving in, let's take a quick look at brim. 



#### Brim desktop app   


<p></p>

Brim is an open-source / cross-platform desktop application (electron, nodeJS, react) that uses Zeek to generate logs from ingested PCAPs. The logs are then combined into [ZNG](https://github.com/brimsec/zq/blob/master/zng/docs/spec.md) format and displayed within the app for consumption by the analyst. Brim uses zq and zqd on the back-end to handle the search, analysis and transformation of the PCAPs into structured logs. zq evaluates [ZQL](https://github.com/brimsec/zq/blob/master/zql/docs/README.md) queries against input log files, producing output log streams in the ZNG format. ZQD serves a RESTful API used to manage and query spaces that contain log data. 

Here is a visual breakdown:

![brim-architecture](/images/mrnatural/brim.jpg)

For a deeper dive, check out [Brim Overview for Developers](https://www.youtube.com/watch?v=CPel0iu1pig). Now let's dive into the challenge. 

#### Mr. Natural 

We are going to have to tackle these questions in non-sequential order, as I cannot identify the IP / Hostname / User of the infected machine, until well, I identify the actual infection. Getting started, we have lots of information from Brad when  he broke down the environment and from my experience the most likely method will be an EXE dropped from HTTP. Let's load the PCAP into brim and see what we can find!

Once you drag and drop the PCAP into brim, you are presented with the search page and the results from the import Let's do an exploratory query defining the path as http as follows:

```_path=http | sort ts```

![_path http search](/images/mrnatural/brim-search-http.jpg)

If you sort ts (timestamp field) then the connection that jumps out is the first one. The id.resp_h is 5.44.43.72 (t497sword.com) and id.orig_h 10.12.1.101. Let's dive into more details by bringing up the log details.  We see that the mime_type reported by ```zeek``` is ```application/x-dosexec```. We can drill down even further from here, by navigating to the ```files path```. We are now presented with hashes (md5/sha1) as well as tx_host (transmit) and rx_host (recipient). We also get correlation of the UID that is generated by zeek across all the logs. We could keep digging deeper and extrac the file, but there is no need for that for our requirements and end game. Right-clicking the md5 hash allows us to submit a query to VT and see what we get. Just as suspected, it is being identified by 45 / 68 engines. It appears to e a variant of Razy or GenKryptik. It looks like with just this identification, we can answer several questions as follows:

#### What is the date and time of the infection?

We can say that UTC timestamp of 23:41:43 on 12-01-2020 is the start of this infection. Which is the corresponding timestamp of the GET request of the malicious executable. 

#### What is the IP of the Infected Windows Host?


The IP is 10.12.1.101 as seen in the details of the download of the malicious executable from 5.44.43.72 (t497sword.com). 

##### What is the SHA256 hash of the EXE downloaded from 5.44.43.72?

Once we sent the request to VirusTotal, we can check under the ```details``` tab which will break down everything about the file it observed. 


| Attr  | Value |
|-------- |:---------:|
| File type: | ```win32.dll``` |
| File size: | ```215552 bytes``` |
| SHA256: |```	f77657e1341bee58750948e1d7ea50b052ee624937144d497787967f5f422e7f```|
| MD5: | ```17b50c1da7d23d686fccfa8de3d27a3a``` |


Moving on, let's try and identify the username, hostname and MAC address since we now know the IP of the infected workstation. We know that this is a Windows environment with a DC (Domain Controller). We know that LDAP uses kerberos tikets for authentication, we can specify a query settig the path to kerberos:

```_path=kerberos```

Once the results are in, we fidnd the information we are looking for under the ```client``` column. 

![kerberos username](/images/mrnatural/krb5-user.jpg)

And there we go, the username on the infected host was ```fabulous.dave```. Within those same results, we can also answer the question: 

#### What is the hostname and MAC of the Infected Host?

The hostname appears to be ```MRNATURAL-DESKTOP```. FQDN MRNATURAL-DESKTOP.mrnatural.local. To retrieve the MAC address, I had to fire up the pcap in Wireshark, as zeek does not really include Layer 2 information in its log files. 

Opeming the PCAP in wireshark, it was really straight forward, we can get the MAC from the first request (IGMP) and the corresponding MAC: ```00:05:5e:3b:42:8c```. That pretty much sums up the analysis piece when it comes to the infection and affected hosts. The only thing left to piece together is the SSL question. 

##### Which two IP addresses and associated domains have HTTPS traffic with "Internet Widgets Pty" as part of the certificate data?

With Brim this is also fairly straight forward and easy to accomplish with the fact that the ZQL queries can do sort, uniq, etc. This is easily accomplished with a simple query as follows:

```_path=ssl subject="O=Internet Widgits Pty Ltd,ST=Some-State,C=AU,CN=localhost" | cut server_name | sort | uniq```

will yield: 

![server_names for ssl certs](/images/mrnatural/ssl.jpg)

And that is it! We have answered all the questions posed by Brad, and we did it all with Brim. I cannot wait until I get much more proficient with this tool and its search syntax because I have been impressed thus far. For the next go round, I will accomplish answering the PCAP channelge but using zq on the command-line to showcase it and learn it as well. 

Until next time. 
