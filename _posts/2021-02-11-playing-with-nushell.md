overflo---
title: "Playing with Nushell"
date: 2021-02-11  16:32:00
categories: [nushell, rust, command line-fu]
tags: [nushell, command line ninjitsu, analysis]
---

<a href="http://www.nushell.sh"><img src="/images/nushell/nushell.png" style="border-radius:50%;margin:20px 30px" ALIGN="left" height="100" width="100"> </a>
I recently came across [Nushell](http://github.com/nushell/nushell). It bills itself as a modern shell for the github era built on Rust. Drawing inspiration from Powershell and other functional programming languages it aims to treat files and input as structured data  It is fairly new software as the current version is 0.26.0. There are a number of ways to go about installing Nu shell, with packages available for the major distributions as well as a docker image you can pull from docker-hub. While I do not see myself changing my default shell over to nu, based on some of the features I will be using it during my various analysis tasks that are not automated. I decided to use cargo to build plugins from source using [crates](http://crates.io). The pre-packaged rpm's for Fedora only included the nu-core and its required dependecies. I choose the plugins listed below to include in my environment. Thanks to the active community on Discord (Thanks @jtuner) I had to make sure that all nu plugin binaries are located within the same directory as the nu binary for them to be available within the nu environment, regardless if they were in my PATH or not. I symlinked the binaries to ~/.cargo/bin and off we go! 

  - nu_plugin_chart
  - nu_plugin_tree
  - nu_plugin_to_sqlite
  - nu_plugin_from_sqlite
  - nu_plugin_match
  - nu_plugin_post
  - nu_plugin_histogram


In Unix, piping commads together to combine several steps into a sophisticated command line executiion is common place. Nu takes this a step further with the idea of pipelines. Nu allows for commands to output from STDOUT and read from STDIN. As I stated prior, nu attempts to treat output from commands and lifes as semi-structured data. It will represent that data as such when handing the output from an invoked command, such as:

{% highlight bash %}

/home/electr0n/analysis/data> sys | get disks
───┬─────────────────────────┬──────┬───────────┬──────────┬──────────
 # │         device          │ type │   mount   │  total   │   free
───┼─────────────────────────┼──────┼───────────┼──────────┼──────────
 0 │ /dev/mapper/fedora-root │ ext4 │ /         │  52.6 GB │   7.7 GB
 1 │ /dev/nvme0n1p2          │ ext4 │ /boot     │   1.0 GB │ 701.1 MB
 2 │ /dev/nvme0n1p1          │ vfat │ /boot/efi │ 209.5 MB │ 188.2 MB
 3 │ /dev/mapper/fedora-home │ ext4 │ /home     │ 182.7 GB │ 110.9 GB
───┴─────────────────────────┴──────┴───────────┴──────────┴──────────

{% endhighlight %}
 
Here we called an internal nu command `sys` and using `get` another internal command, we can select on the disks column. If we had not used the `get` command we would have been greeted with output similar to:

{% highlight bash %}

/home/electr0n/analysis/data> sys
───┬─────────────────────────────────────────────┬────────────────┬────────────────┬───────────────────────────────────────┬────────────────┬────────────────
 # │                    host                     │      cpu       │     disks      │                  mem                  │      temp      │      net       
───┼─────────────────────────────────────────────┼────────────────┼────────────────┼───────────────────────────────────────┼────────────────┼────────────────
 0 │ [row name version hostname uptime sessions] │ [table 4 rows] │ [table 4 rows] │ [row total free swap total swap free] │ [table 7 rows] │ [table 6 rows] 
───┴─────────────────────────────────────────────┴────────────────┴────────────────┴───────────────────────────────────────┴────────────────┴────────────────

{% endhighlight %}

The 4 rows under `cpu` column corresponds with the 4 cores of my CPU. The `net` column will correspond to the 6 interface

{% highlight bash %}

/home/electr0n/analysis/data> ls | where size > 10MB
───┬──────────────────┬──────┬──────────┬──────────────
 # │       name       │ type │   size   │   modified   
───┼──────────────────┼──────┼──────────┼──────────────
 0 │ UNSW-NB15_2.csv  │ File │ 165.2 MB │ 2 hours ago  
 1 │ eduroam-ddos.csv │ File │  44.1 MB │ 7 months ago 
 2 │ eduroam.csv      │ File │  16.4 MB │ 7 months ago 
 3 │ jre-overflow.csv │ File │  14.7 MB │ 7 months ago 
 4 │ nitroba.csv      │ File │  17.2 MB │ 7 months ago 
───┴──────────────────┴──────┴──────────┴──────────────
{% endhighlight %}

The way Nu formats the output is appealing to me as well as being able to manipulate the data with commands that seem familiar and straight forward is extremely appealing. You can type help to get a full listing of commands available to you. You can also use native commands within 
the nu environment. Let's take it for a spin and use it to 'play' with some data. I have chosen 

{% highlight bash %}

/home/electr0n/analysis/data> open honeypots.alerts.json | first 9

───┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬───────┬────────────────┬──────────┬────────────────────────────
 # │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │ proto │     src_ip     │ src_port │         timestamp          
───┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼───────┼────────────────┼──────────┼────────────────────────────
 0 │ [row 7 columns] │ 138.68.3.71 │        22 │ alert      │ 52710912 │ eth0     │ TCP   │ 8.42.77.171    │    49678 │ 2019-01-02T03:50:11.315110 
 1 │ [row 7 columns] │ 138.68.3.71 │      5811 │ alert      │ 52589280 │ eth0     │ TCP   │ 8.42.77.171    │    49306 │ 2019-01-02T03:50:10.621656 
 2 │ [row 7 columns] │ 138.68.3.71 │      5915 │ alert      │ 52491840 │ eth0     │ TCP   │ 8.42.77.171    │    65386 │ 2019-01-02T03:50:10.386108 
 3 │ [row 7 columns] │ 138.68.3.71 │      5060 │ alert      │ 53053968 │ eth0     │ UDP   │ 80.211.246.121 │     5130 │ 2019-01-02T03:53:24.798186 
 4 │ [row 7 columns] │ 138.68.3.71 │      1433 │ alert      │ 52568784 │ eth0     │ TCP   │ 8.42.77.171    │    49238 │ 2019-01-02T03:50:10.576769 
 5 │ [row 7 columns] │ 138.68.3.71 │      1521 │ alert      │ 52576512 │ eth0     │ TCP   │ 8.42.77.171    │    49269 │ 2019-01-02T03:50:10.585758 
 6 │ [row 7 columns] │ 138.68.3.71 │      3306 │ alert      │ 52373568 │ eth0     │ TCP   │ 8.42.77.171    │    65036 │ 2019-01-02T03:50:09.097718 
 7 │ [row 7 columns] │ 138.68.3.71 │      5432 │ alert      │ 52507296 │ eth0     │ TCP   │ 8.42.77.171    │    65438 │ 2019-01-02T03:50:10.421359 
 8 │ [row 7 columns] │ 138.68.3.71 │      5060 │ alert      │ 53055648 │ eth0     │ UDP   │ 37.49.231.178  │     7433 │ 2019-01-02T03:54:44.259003 
───┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴───────┴────────────────┴──────────┴────────────────────────────


{% endhighlight %}

We are able to load this data because JSON is a supported data-type. For a complete list, please refer to [Loading Data](https://www.nushell.sh/book/loading_data.html). We could have also substituted `first` with `range 1..9` to retrieve the same table data. Right away we can see that the alert column will take us deeper into the nested JSON because it is reporting that that row will have a further 7 columns to display. Let's clean this up a bit more by sorting the output even further. We will use another internal command, `sort-by` and specifically the src_ip column:

{% highlight bash %}

/home/electr0n/analysis/data> open honeypots.alerts.json | sort-by 'src_ip'
───┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬─────────┬───────┬──────────────┬──────────┬────────┬────────────────────────────
 # │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │ payload │ proto │    src_ip    │ src_port │ stream │         timestamp          
───┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼─────────┼───────┼──────────────┼──────────┼────────┼────────────────────────────
 0 │ [row 7 columns] │ 138.68.3.71 │      8181 │ alert      │ 43593424 │ eth0     │         │ TCP   │ 1.169.38.217 │    37667 │      0 │ 2019-01-02T06:50:03.088003 
───┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴─────────┴───────┴──────────────┴──────────┴────────┴────────────────────────────
───┬─────────────────┬────────────────┬───────────┬────────────┬──────────┬─────────────────┬──────────┬─────────────────────────────┬─────────────────────────────┬───────┬─────────────┬──────────┬────────┬────────────────────────────
 # │      alert      │    dest_ip     │ dest_port │ event_type │ flow_id  │      http       │ in_iface │           payload           │      payload_printable      │ proto │   src_ip    │ src_port │ stream │         timestamp          
───┼─────────────────┼────────────────┼───────────┼────────────┼──────────┼─────────────────┼──────────┼─────────────────────────────┼─────────────────────────────┼───────┼─────────────┼──────────┼────────┼────────────────────────────
 1 │ [row 8 columns] │ 198.199.99.226 │        80 │ alert      │ 43586704 │ [row 8 columns] │ eth0     │ 0LXXcHd/AAAAEAAAmQAAAEdFVCA │ GET                         | TCP   │ 138.68.3.71 │  41694   │   1    │ 2019-01-02T06:42:52.658017 
   │                 │                │           │            │          │                 │          │ vdWJ1bnR1L3Bvb2wvbWFpbi90L3 │ /ubuntu/pool/main/t/tzdata/ │       │             │          │        │                            
   │                 │                │           │            │          │                 │          │ R6ZGF0YS90emRhdGFfMjAxOGktM │ tzdata_2018i-0ubuntu0.16.04 │       │             │          │        │                            
   │                 │                │           │            │          │                 │          │ HVidW50dTAuMTYuMDRfYWxsLmRl │ _all.deb                    │       │             │          │        │                            
   |                 │                │           │            │          │                 │          │ YiBIVFRQLzEuMQ0KSG9zdDogbWl │ HTTP/1.1                    |       |             |          |        |
   │                 │                │           │            │          │                 │          │ ycm9ycy5kaWdpdGFsb2NlYW4uY2 │ Host:                       │       │             │          │        │                            
   │                 │                │           │            │          │                 │          │ 9tDQpVc2VyLUFnZW50OiBEZWJpY │ mirrors.digitalocean.com    |       |             |          |        |
   │                 │                │           │            │          │                 │          │ W4gQVBULUhUVFAv             │ User-Agent: Debian          │       │             │          │        │                            
   │                 │                │           │            │          │                 │          │ APT-HTTP/1.3 (1.2.29)       |                             |       |             |          |        | 
   |                 |                │           │            │          │                 │          │                             │                             |       |             |          |        |
───┴─────────────────┴────────────────┴───────────┴────────────┴──────────┴─────────────────┴──────────┴─────────────────────────────┴─────────────────────────────┴───────┴─────────────┴──────────┴────────┴────────────────────────────
───┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬───────┬────────────────┬──────────┬────────────────────────────
 # │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │ proto │     src_ip     │ src_port │         timestamp          
───┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼───────┼────────────────┼──────────┼────────────────────────────
 2 │ [row 7 columns] │ 138.68.3.71 │       161 │ alert      │ 53789472 │ eth0     │ UDP   │ 184.105.139.67 │    58695 │ 2019-01-02T04:48:08.177654 
 3 │ [row 7 columns] │ 138.68.3.71 │        23 │ alert      │ 53777040 │ eth0     │ TCP   │ 185.244.25.145 │    55534 │ 2019-01-02T04:35:53.007299 
───┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴───────┴────────────────┴──────────┴────────────────────────────
───┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬──────────────┬───────────────────┬───────┬────────────────┬──────────┬────────┬────────────────────────────
 # │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │   payload    │ payload_printable │ proto │     src_ip     │ src_port │ stream │         timestamp          
───┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼──────────────┼───────────────────┼───────┼────────────────┼──────────┼────────┼────────────────────────────
 4 │ [row 7 columns] │ 138.68.3.71 │       123 │ alert      │ 43527568 │ eth0     │ Li4uKi4uLi4u │ ...*.....         │ UDP   │ 185.244.25.167 │    43991 │      0 │ 2019-01-02T05:43:18.253590 
───┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴──────────────┴───────────────────┴───────┴────────────────┴──────────┴────────┴────────────────────────────
───┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬─────────┬───────┬────────────────┬──────────┬────────┬────────────────────────────
 # │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │ payload │ proto │     src_ip     │ src_port │ stream │         timestamp          
───┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼─────────┼───────┼────────────────┼──────────┼────────┼────────────────────────────
 5 │ [row 7 columns] │ 138.68.3.71 │        23 │ alert      │ 43523200 │ eth0     │         │ TCP   │ 31.163.158.140 │    31870 │      0 │ 2019-01-02T05:37:47.854589 
 6 │ [row 7 columns] │ 138.68.3.71 │     33098 │ alert      │ 43513456 │ eth0     │         │ TCP   │ 31.192.108.68  │    53597 │      0 │ 2019-01-02T05:27:44.153921 
 7 │ [row 7 columns] │ 138.68.3.71 │      8030 │ alert      │ 43512784 │ eth0     │         │ TCP   │ 37.49.231.125  │    22138 │      0 │ 2019-01-02T05:26:24.432721 
───┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴─────────┴───────┴────────────────┴──────────┴────────┴────────────────────────────
────┬─────────────────┬─────────────┬───────────┬────────────┬──────────┬──────────┬───────┬────────────────┬──────────┬────────────────────────────
 #  │      alert      │   dest_ip   │ dest_port │ event_type │ flow_id  │ in_iface │ proto │     src_ip     │ src_port │         timestamp          
────┼─────────────────┼─────────────┼───────────┼────────────┼──────────┼──────────┼───────┼────────────────┼──────────┼────────────────────────────
  8 │ [row 7 columns] │ 138.68.3.71 │      5060 │ alert      │ 53055648 │ eth0     │ UDP   │ 37.49.231.178  │     7433 │ 2019-01-02T03:54:44.259003 
  9 │ [row 7 columns] │ 138.68.3.71 │      5060 │ alert      │ 53055648 │ eth0     │ UDP   │ 37.49.231.178  │     7433 │ 2019-01-02T03:54:44.259003 
 10 │ [row 7 columns] │ 138.68.3.71 │      1433 │ alert      │ 53398368 │ eth0     │ TCP   │ 43.227.231.129 │    48703 │ 2019-01-02T03:56:08.277797 
 11 │ [row 7 columns] │ 138.68.3.71 │      1433 │ alert      │ 44288960 │ eth0     │ TCP   │ 45.62.211.169  │    45638 │ 2019-01-02T05:15:32.504812 
────┴─────────────────┴─────────────┴───────────┴────────────┴──────────┴──────────┴───────┴────────────────┴──────────┴────────────────────────────

.....[ truncated ]........

{% endhighlight %}

As you may have noticed, the index starts the count at zero, like you would see in JS and other programming languages. Entry #1 (2nd) shows a new column previously not seen and that is `http`. We see that if we were to dive into 
that field we would be presented with 1 row and 8 column's. If we just add a get to the pipeline, it would throw an error as the first line (# 0) does not have a column labeled http. Here we can use the `skip` command and then invoke `get` on the http
column like so:

{% highlight bash %}

/home/electr0n/analysis//data/> open honeypots.alerts.json | sort-by 'src_ip' | skip 1 | get http
───┬──────────────────────────┬──────────────────────────┬─────────────┬──────────────────────────────┬────────┬──────────┬────────┬────────────────────────────────────────────────────────────────
 # │         hostname         │    http_content_type     │ http_method │       http_user_agent        │ length │ protocol │ status │                              url                               
───┼──────────────────────────┼──────────────────────────┼─────────────┼──────────────────────────────┼────────┼──────────┼────────┼────────────────────────────────────────────────────────────────
 0 │ mirrors.digitalocean.com │ application/octet-stream │ GET         │ Debian APT-HTTP/1.3 (1.2.29) │   1197 │ HTTP/1.1 │ 200    │ /ubuntu/pool/main/t/tzdata/tzdata_2018i-0ubuntu0.16.04_all.deb 
───┴──────────────────────────┴──────────────────────────┴─────────────┴──────────────────────────────┴────────┴──────────┴────────┴────────────────────────────────────────────────────────────────
{% endhighlight %}

Right after correctly retrieving the data from the row and subsequent columns' it will throw an error since it wants to continue to process that column as it works thru the rest of the loaded data. I attempted to see if we could pass a `range 1..1` to the pipeline
to only retrieve that entry, and it worked quite nicely. From the actual signature we can see that category was non-suspicious traffic and the url appears to back the claim up. APT user-agent downloading a debian .deb package. Let's see how easy it is to pull out all signatures 
that are housed within this json alerts file. This is fairly straight forward and only requires the addition of 1 more command to the pipeline. Of course we want to `get alert` and get a full list. 

{% highlight bash %}

home/electr0n/analysis//data/> open honeypots.alerts.json | sort-by 'src_ip' | get alert
───┬─────────┬─────────────┬─────┬───────┬──────────┬───────────────────────────────────────────────────────────────────┬──────────────
 # │ action  │  category   │ gid │  rev  │ severity │                             signature                             │ signature_id 
───┼─────────┼─────────────┼─────┼───────┼──────────┼───────────────────────────────────────────────────────────────────┼──────────────
 0 │ allowed │ Misc Attack │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 1 │      2403300 
───┴─────────┴─────────────┴─────┴───────┴──────────┴───────────────────────────────────────────────────────────────────┴──────────────
───┬─────────┬────────────────────────┬─────┬─────┬──────────┬──────────────────────────────────────────────────────────────────────────────────┬──────────────┬───────
 # │ action  │        category        │ gid │ rev │ severity │                                    signature                                     │ signature_id │ tx_id 
───┼─────────┼────────────────────────┼─────┼─────┼──────────┼──────────────────────────────────────────────────────────────────────────────────┼──────────────┼───────
 1 │ allowed │ Not Suspicious Traffic │   1 │   3 │        3 │ ET POLICY GNU/Linux APT User-Agent Outbound likely related to package management │      2013504 │     0 
───┴─────────┴────────────────────────┴─────┴─────┴──────────┴──────────────────────────────────────────────────────────────────────────────────┴──────────────┴───────
────┬─────────┬────────────────────────────┬─────┬───────┬──────────┬───────────────────────────────────────────────────────────────────────┬──────────────
 #  │ action  │          category          │ gid │  rev  │ severity │                               signature                               │ signature_id 
────┼─────────┼────────────────────────────┼─────┼───────┼──────────┼───────────────────────────────────────────────────────────────────────┼──────────────
  2 │ allowed │ Attempted Information Leak │   1 │    12 │        2 │ GPL SNMP public access udp                                            │      2101411 
  3 │ allowed │ Misc Attack                │   1 │  4934 │        2 │ ET COMPROMISED Known Compromised or Hostile Host Traffic TCP group 46 │      2500090 
  4 │ allowed │ Misc Attack                │   1 │  5047 │        2 │ ET DROP Dshield Block Listed Source group 1                           │      2402001 
  5 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 18    │      2403334 
  6 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 19    │      2403336 
  7 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 24    │      2403346 
  8 │ allowed │ Attempted Information Leak │   1 │     4 │        2 │ ET SCAN Sipvicious User-Agent Detected (friendly-scanner)             │      2011716 
  9 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP UDP group 24    │      2403347 
 10 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 32    │      2403362 
 11 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 35    │      2403368 
 12 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 36    │      2403370 
 13 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 3     │      2403304 
 14 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 4     │      2403306 
 15 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 5     │      2403308 
 16 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 44    │      2403386 
 17 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 45    │      2403388 
 18 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 49    │      2403396 
 19 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 56    │      2403410 
 20 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 57    │      2403412 
 21 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 62    │      2403422 
 22 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP UDP group 67    │      2403433 
 23 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 67    │      2403432 
 24 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 73    │      2403444 
 25 │ allowed │ Attempted Information Leak │   1 │    19 │        2 │ ET SCAN Potential SSH Scan                                            │      2001219 
 26 │ allowed │ Attempted Information Leak │   1 │     5 │        2 │ ET SCAN Potential VNC Scan 5800-5820                                  │      2002910 
 27 │ allowed │ Attempted Information Leak │   1 │     5 │        2 │ ET SCAN Potential VNC Scan 5900-5920                                  │      2002911 
 28 │ allowed │ Potentially Bad Traffic    │   1 │     3 │        2 │ ET SCAN Suspicious inbound to MSSQL port 1433                         │      2010935 
 29 │ allowed │ Potentially Bad Traffic    │   1 │     3 │        2 │ ET SCAN Suspicious inbound to Oracle SQL port 1521                    │      2010936 
 30 │ allowed │ Potentially Bad Traffic    │   1 │     3 │        2 │ ET SCAN Suspicious inbound to mySQL port 3306                         │      2010937 
 31 │ allowed │ Potentially Bad Traffic    │   1 │     3 │        2 │ ET SCAN Suspicious inbound to PostgreSQL port 5432                    │      2010939 
 32 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 76    │      2403450 
 33 │ allowed │ Attempted Information Leak │   1 │     6 │        2 │ ET SCAN Sipvicious Scan                                               │      2008578 
 34 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP UDP group 76    │      2403451 
 35 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 77    │      2403452 
 36 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 79    │      2403456 
 37 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 86    │      2403470 
 38 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 87    │      2403472 
 39 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP UDP group 89    │      2403477 
 40 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 92    │      2403482 
 41 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 93    │      2403484 
 42 │ allowed │ Misc Attack                │   1 │  5047 │        2 │ ET DROP Dshield Block Listed Source group 1                           │      2402000 
 43 │ allowed │ Misc Attack                │   1 │ 46061 │        2 │ ET CINS Active Threat Intelligence Poor Reputation IP TCP group 94    │      2403486 
────┴─────────┴────────────────────────────┴─────┴───────┴──────────┴───────────────────────────────────────────────────────────────────────┴──────────────
/home/electr0n/analysis//data/> 

{% endhighlight %}

That was quick and easy to navigate the multiple levels of the nested JSON alerts file. Let's check out another scenario with the same dataset, when you `group-by` src_ip versus `sort-by`. 
The resulting table is hard to read, but thankfully nushell provides a simple and effective command to handle the flipping of rows and columns. Shown in the following output. 

{% highlight bash %}

/home/electr0n/analysis//data/> open honeypots.alerts.json | group-by 'src_ip'
───┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────
 # │ 8.4 │ 80. │ 37. │ 138 │ 184 │ 92. │ 185 │ 1.1 │ 5.1 │ 5.1 │ 5.1 │ 31. │ 31. │ 37. │ 43. │ 45. │ 46. │ 58. │ 58. │ 59. │ 61. │ 61. │ 66. │ 71. │ 71. │ 78. │ 80. │ 80. │ 82. │ 88. │ 88. │ 89. │ 91. │ 92. │ 185 
   │ 2.7 │ 211 │ 49. │ .68 │ .10 │ 63. │ .24 │ 69. │ 01. │ 41. │ 88. │ 163 │ 192 │ 49. │ 227 │ 62. │ 100 │ 218 │ 48. │ 47. │ 176 │ 219 │ 240 │ 6.2 │ 6.1 │ 128 │ 211 │ 82. │ 202 │ 202 │ 214 │ 248 │ 245 │ 53. │ .24 
   │ 7.1 │ .24 │ 231 │ .3. │ 5.1 │ 194 │ 4.2 │ 38. │ 40. │ 67. │ 206 │ .15 │ .10 │ 231 │ .23 │ 211 │ .43 │ .21 │ 32. │ 71. │ .22 │ .11 │ .23 │ 32. │ 99. │ .11 │ .17 │ 77. │ .20 │ .19 │ .26 │ .16 │ .36 │ 76. │ 4.2 
   │ 71  │ 6.1 │ .17 │ 71  │ 39. │ .33 │ 5.1 │ 217 │ 81  │ 237 │ .22 │ 8.1 │ 8.6 │ .12 │ 1.1 │ .16 │ .82 │ 3.6 │ 199 │ 13  │ 2.1 │ .15 │ 6.1 │  4  │ 23  │ 2.9 │ 2.2 │ 33  │ 9.1 │ 0.1 │ .54 │ 8.5 │ .26 │ 213 │ 5.1 
   │     │ 21  │  8  │     │ 67  │     │ 67  │     │     │     │     │ 40  │  8  │  5  │ 29  │  9  │     │     │     │     │ 67  │  1  │ 19  │     │     │  8  │ 33  │     │ 55  │ 48  │     │  1  │     │     │ 45  
───┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────┼─────
 0 │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta │ [ta 
   │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble │ ble 
   │ 7   │ 2   │ 2   │ 1   │ 1   │ 2   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   │ 1   
   │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row │ row 
   │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  │ s]  
───┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────

{% endhighlight %}

As you can see, the column names are now IP addresses and would be cumbersome to try and iterate through these entries. Utilizing Nu's built-in `pivot` command, we are able to flip this around and be presented with the output as:

{% highlight bash %}

/home/electr0n/analysis//data/> open honeypots.alerts.json | group-by 'src_ip' | pivot SRC_IP Alert
────┬────────────────┬────────────────
 #  │    SRC_IP      │    Alert     
────┼────────────────┼────────────────
  0 │ 8.42.77.171    │ [table 7 rows] 
  1 │ 80.211.246.121 │ [table 2 rows] 
  2 │ 37.49.231.178  │ [table 2 rows] 
  3 │ 138.68.3.71    │ [table 1 rows] 
  4 │ 184.105.139.67 │ [table 1 rows] 
  5 │ 92.63.194.33   │ [table 2 rows] 
  6 │ 185.244.25.167 │ [table 1 rows] 
  7 │ 1.169.38.217   │ [table 1 rows] 
  8 │ 5.101.40.81    │ [table 1 rows] 
  9 │ 5.141.67.237   │ [table 1 rows] 
 10 │ 5.188.206.22   │ [table 1 rows] 
 11 │ 31.163.158.140 │ [table 1 rows] 
 12 │ 31.192.108.68  │ [table 1 rows] 
 13 │ 37.49.231.125  │ [table 1 rows] 
 14 │ 43.227.231.129 │ [table 1 rows] 
 15 │ 45.62.211.169  │ [table 1 rows] 
 16 │ 46.100.43.82   │ [table 1 rows] 
 17 │ 58.218.213.6   │ [table 1 rows] 
 18 │ 58.48.32.199   │ [table 1 rows] 
 19 │ 59.47.71.13    │ [table 1 rows] 
 20 │ 61.176.222.167 │ [table 1 rows] 
 21 │ 61.219.11.151  │ [table 1 rows] 
 22 │ 66.240.236.119 │ [table 1 rows] 
 23 │ 71.6.232.4     │ [table 1 rows] 
 24 │ 71.6.199.23    │ [table 1 rows] 
 25 │ 78.128.112.98  │ [table 1 rows] 
 26 │ 80.211.172.233 │ [table 1 rows] 
 27 │ 80.82.77.33    │ [table 1 rows] 
 28 │ 82.202.209.155 │ [table 1 rows] 
 29 │ 88.202.190.148 │ [table 1 rows] 
 30 │ 88.214.26.54   │ [table 1 rows] 
 31 │ 89.248.168.51  │ [table 1 rows] 
 32 │ 91.245.36.26   │ [table 1 rows] 
 33 │ 92.53.76.213   │ [table 1 rows] 
 34 │ 185.244.25.145 │ [table 1 rows] 
────┴────────────────┴────────────────

{% endhighlight %}

Not only were we able to flip columns and rows but instead of using the default Column0 / Column1 headings I am able to give custom column labels without doing anything fancy. I really like this feature. This also works when working with CSV's.  Instantly we have actionable data at Row 0. At this point you should be able to continue to dig as deep as you'd like to gain the insight to resolve the issue. Nushell has also implemented basic charting via lines and bar graphs into the environment. A simple bar chart example using 1 column is:

![nu-shell-graph](/images/nushell/nu-plugin-chart.jpg)

there are too many features to dive into just for this post and I feel like we have been able to show some strengths that nushell brings. It will be nice when `from url` can handle 'text/javascript', but as of right now it cannot. I did not really play with the `post` plugin, but if you do, reach out and let me know! Even though nushell is still in its infancy, currently has a lot to offer and the future is bright. While it would take a lot to move me away from my current zsh setup, none the less I will be using it often since I do like the way it formats stdout into structured tables, allowing for quick analysis and parsing. 

