---
title: "On the wire: SSH Tunnels"
date: 2021-04-06 02:26:00 
categories: [ covert, tunnels, reverse shells, detection, fingerprinting]
tags: [fingerprinting, tunnels, shells, detection, shells]
---

This blog will provide an introduction into the SSH protocol handshake and how it pertains to the identification and detection of SSH tunnels.
Then we will introduce a method to fingerprint individual SSH clients and servers and ultimately end with detection of SSH tunneling using both
the standalone python version and the zeek implementation of packetStrider. 

# SSH Tunnels 

The Secure SHell protocol is heavily used across most enterprise environments. Observing ssh traffic in your environment likely is commonplace. 
The most widely used implementation of the secure shell protocol is OpenSSH, which can be leveraged to create l2/l3 tunnels using tun interfaces, 
dynamic port forwarding (SOCKS) proxy, or local / remote port-forwarding tunnels. SSH is also heavily employed by threat groups during various
phases of a campaigns such as data exfiltration, maintaining persistence and tunneling. 

![ssh tunnel illustration](/images/tunnels/ssh-tunnels-ex.png)
_Figure 1: high-level depiction_


### -D dynamic forward

This works by allocating a socket to listen to port on the local side, optionally bound to the specified
bind_address.  Whenever a connection is made to this port, the connection is forwarded over the secure channel, and the application protocol is then used to determine where to
connect to from the remote machine.  Currently the SOCKS4 and SOCKS5 protocols are supported, and ssh will act as a SOCKS server.  

### -R remote forward

![remote tunnel](/images/tunnels/SSH_Reverse_Tunnel.png)
_Figure 2: Reverse Tunnels_

This works by allocating a socket to listen to either a TCP port or to a Unix socket on the remote side.  Whenever a connection is made to this port or Unix socket, the connection is forwarded
over the secure channel, and a connection is made from the local machine to either an explicit destination specified by host port hostport, or local_socket, or, if no explicit destination was
specified, ssh will act as a SOCKS 4/5 proxy and forward connections to the destinations requested by the remote SOCKS client.

### -L local forward 

ssh -L 

Specifies that the given port on the local (client) host is to be forwarded to the given host and port on the remote side of the ssh connection. This works by allocating a socket to listen on that port on the local side, optionally bound to the specified bind_address. Whenever a connection is made to this port, the connection is forwarded thru the secure channel, and a connection is made to host port hostport from the remote machine.

I won't even touch on agent forwarding (ssh -A) as [Skylight Cyber](ttps://skylightcyber.com/2019/09/26/all-your-cloud-are-belong-to-us-cve-2019-12491/) did a great job around that. Now that we have a decent introduction let's move onto why you are here. 

## SSH protocol handshake  

The SSH protocol relies on the client and server negotiating parameters during an initial handshake before a secure channel can be established. From a high level this looks like: 

- Protocol version exchange 
- Key exchange
- Elliptic Curve DH Init
- Elliptic DH Curve Reply
- New Key

After protocol version checks the KEX (Key EXchange) process is kicked off by issuing a SSH_MSG_KEX_INIT (server -> client) message with a list of cryptographic primitives supported. 
The other side will also provide a list of their preference. These lists are ordered by preference as when both sides match a supported primitive, it'll be selected. These keys will 
be session based as different clients have unique supported algorithms and order of preference. These exchanges are the initialization for the encrypted channel. Client and Server will 
use these primitives for key exchange, message authentication and ultimately data encryption. 
Per the RFC: [RFC 4253](https://tools.ietf.org/html/rfc4253#section-5.3) 

{% highlight text %}

6.3.  Encryption

   An encryption algorithm and a key will be negotiated during the key
   exchange.  When encryption is in effect, the packet length, padding
   length, payload, and padding fields of each packet MUST be encrypted
   with the given algorithm.

{% endhighlight %}

When invoking a ssh connection with very verbose output from the cli it will output to stdin each process in the ssh handshake, you have seen it before:

```bash
debug1: kex: server->client cipher: aes256-gcm@openssh.com MAC: <implicit> compression: none
debug1: kex: client->server cipher: aes256-gcm@openssh.com MAC: <implicit> compression: none
debug1: kex: curve25519-sha256 need=32 dh_need=32
debug1: kex: curve25519-sha256 need=32 dh_need=32
```

In this example, its linux to linux, aes256-gcm and curve25519-sha256 have been agreed upon, with no compression. Once the server receives SSH_MSG_KEX_ECDH_INIT 
it can then generate it's own Ephemeral keypair, together with the client pub key, generate the shared secret K. 

At this point, the server needs to generate the exchange hash and sign it producing HS. Shown here:

![Generation of exchange hash H](/images/tunnels/kex-ecdh-hash.svg)

All the required criteria is present for the server to generate SSH_MSG_KEX_ECDH_REPLY which is depicted below.

![ECDH KEX Reply](/images/tunnels/kex-ecdh-reply.jpg)

Note that the keypairs are ephemeral and will only be used during the key exchange and discarded afterwards. These lists of ciphers and hash algorithm 
can be unique enough to be used as a potential fingerprint towards identification of different client and server implementations. 


### Client/Server fingerprinting

A recent report by Recorded Future titled; 
[Adversary Infrastructure Report 2020: A Defender's View](https://go.recordedfuture.com/hubfs/reports/cta-2021-0107.pdf) 
shows threat actors are using readily available opensource frameworks. Per the report, the most prolific c2 families:


| Family        | Count |
|-----------------------|
| Cobalt Strike | 1441  |
| Metasploit    | 1122  |
| PupyRAT       | 454   |
|-----------------------|

The list of top offenssive security tools observed is populated by the top open source projects in 
offenssive security. This can be advantageous for the hunter as lots of research and expertise has gone into detecting these OFST (Opensource Offensive 
Security Tools). Case in point, we have available to us, [Saleforce's HASSH](https://github.com/salesforce/hassh) developed 
by [@benreardon](https://twitter.com/benreardon) which has been generously released as open-source. 

How does HASSH work and what can it help solve/answer? Per the README:

> Detect covert exfiltration of data within the components of the Client algorithm sets. In this case, a specially coded SSH Client can send data outbound from a trusted to a less trusted environment within a series of SSH_MSG_KEXINIT packets. 
> In a scenario similar to the more known exfiltration via DNS, data could be sent as a series of attempted, but incomplete and unlogged connections to an SSH server controlled by bad actors who can then record, decode and reconstitute these 
> pieces of data into their original form. Until now such attempts - much less the contents of the clear text packets - are not logged even by mature packet analyzers or on end point systems. Detection of this style of exfiltration can now be 
> performed easily by using anomaly detection or alerting on SSH Clients with multiple different hassh.

Here are some sample hassh and corresponding tools. 

| hash                             | Client                           |          Tool                |
|----------------------------------------------------------------------------------------------------|
| fafc45381bfde997b6305c4e1600f1bf | Ruby/Net::SSH_5.0.2 x86_64-linux | Metasploit exploit module    |
| b5752e36ba6c5979a575e43178908adf | Python Paramiko_2.4.1 | Metasploit exploit module               |
| de30354b88bae4c2810426614e1b6976 | Powershell Renci.SshNet.SshClient.0.0.1 | Empire exploit        |
| d461cb26d9efc16067d3ff7704cea87f | PupRAT Python                             | PupyRAT             |
|----------------------------------------------------------------------------------------------------|

As we previously discussed, the fact that the list of supported cryptographic primitives (and its order of preference) can be quite unique across different SSH client/Server implementations allows for potential fingerprinting of both client and server. 
Hassh works by constructing an MD5 hash from the specific set of algorithms that are supported by various SSH Client and Servers alike. If hassh has the ability to see the actual SSH_MSG_KEX_INIT handshake. Hassh is not concerned with higher-level ostensible 
identifiers such as client / server version exchanges. Let's use an example straight from the readme: 

#### Client fingerprinting

###### "Cyberduck" SFTP client (specifically SSH-2.0-Cyberduck/6.7.1.28683 (Mac OS X/10.13.6) (x86_64)

|Function|Algorithms for SSH_MSG_KEXINIT|
| ------------- | ------------- |
|Key Exchange methods|`curve25519-sha256@libssh.org,diffie-hellman-group-exchange-sha256,ecdh-sha2-nistp521,ecdh-sha2-nistp384,ecdh-sha2-nistp256,diffie-hellman-group-exchange-sha1,diffie-hellman-group1-sha1,diffie-hellman-group14-sha1,diffie-hellman-group14-sha256,diffie-hellman-group15-sha512,diffie-hellman-group16-sha512,diffie-hellman-group17-sha512,diffie-hellman-group18-sha512,diffie-hellman-group14-sha256@ssh.com,diffie-hellman-group15-sha256,diffie-hellman-group15-sha256@ssh.com,diffie-hellman-group15-sha384@ssh.com,diffie-hellman-group16-sha256,diffie-hellman-group16-sha384@ssh.com,diffie-hellman-group16-sha512@ssh.com,diffie-hellman-group18-sha512@ssh.com`|
|Encryption| `aes128-cbc,aes128-ctr,aes192-cbc,aes192-ctr,aes256-cbc,aes256-ctr,blowfish-cbc,blowfish-ctr,cast128-cbc,cast128-ctr,idea-cbc,idea-ctr,serpent128-cbc,serpent128-ctr,serpent192-cbc,serpent192-ctr,serpent256-cbc,serpent256-ctr,3des-cbc,3des-ctr,twofish128-cbc,twofish128-ctr,twofish192-cbc,twofish192-ctr,twofish256-cbc,twofish256-ctr,twofish-cbc,arcfour,arcfour128,arcfour256`|
|Message Authentication|`hmac-sha1,hmac-sha1-96,hmac-md5,hmac-md5-96,hmac-sha2-256,hmac-sha2-512`|
|Compression|`zlib@openssh.com,zlib,none`|

In an environment which has secure shell tightly defined it would be possible to detect any client as malicious and warrant further investigation that which is outside of your approved clients and their corresponding HASSH MD% signatures. 
You can either use the standalone Python implementation of HASSH or if you are using Zeek, you can deploy it using the pkg manager:

`pkg install hassh`

For more information on utilizing hassh using zeek, please refer to [hassh.zeek readme](https://github.com/salesforce/hassh/tree/master/zeek)

## Detection 

John Althouse III gave a [presentation](https://twitter.com/4A4133/status/855541564067778561) on using Bro to detect ssh tunnels carrying TTY's. 

Through observing packet lengths of keystrokes within attached SSH sessions, insights emerged.

- SSH tunnel + another SSH channel transporting TTY 
  - packet length = SSH Header + [ previous ssh pckt ] + HMAC = either 76, 84, 98 bytes. 

- Exact packet length dependent on block size + HMAC algo impelementations by client/server. 
- Each keystroke is echoed back and can be used to inch towards better accuracy. 

John released some [bro scripts](https://github.com/darkphyber/bro) on github implementing his findings. As well as scripts to hunt for metasploit's default SSL certificate on your network. 

### PacketStrider

[Ben Reardon](http://www.github.com/benjeems/) expanded on his research into SSH. Building upon the previous bro releases by teammate John Althouse, he recently released
[packetstrider](https://www.github.com/benjeems/packetstrider/) as open-source code on his github page. PacketStrider expands on the prior work done with hassh. From the project's 
readme:


`SSH is obviously encrypted, yet valuable contextual information still exists within the network traffic that can go towards TTP's, intent, success and magnitude of actions on objectives. There may even exist situations where valuable context is not available or deleted from hosts, and so having an immutable and un-alterable passive network capture gives additional forensic context. "Packets don't lie"`

`Separately to the forensic context, packet strider predictions could also be used in an active fashion, for example to shun/RST forward connections if a tunneled reverse SSH session initiation feature is predicted within, even before reverse authentication is offered.`

Let's dive into running packetstrider on some pcap's and see what we get. 

{% highlight bash %}

The pcaps used in this example were generated using: `tcpdump -X -s 1514 -w ssh-traffic.pcap -nnvi wlp59s0 'tcp port 22'`


`python3 ~/analysis/tools/packetStrider/python/packetStrider-ssh.py -p -f ~/analysis/data/ssh-traffic.pcap -o .d`

... Loading full pcap : /home/electr0n/analysis/data/ssh-traffic.pcap
... Getting streams from pcap:
    ...found stream 0
    ...found stream 1
    ...found stream 2
    ...found stream 3
... Loading stream 0
... Finding meta
... Finding hassh elements
... Building size matrix

   ... Ordering the keystroke packets
   ... Scanning for Reverse Option being present in forward session init
   ... Scanning for Forward login attempts
   ... Scanning for Forward key accepts
   ... Scanning for Forward login prompts
   ... Scanning for Agent forwarding
   ... Scanning for Reverse Session initiation
   ... Building features with window size = 2, stride = 1
       ... Calculating first packet timestamp
       ... Calculating last packet timestamp
       ... Calculating Max Packet delta
       ... Striding through windows of size 2

┏━━━━ Reporting results for stream 0
┃
┃ Stream 0 of pcap `'/home/electr0n/analysis/data/ssh-traffic.pcap'`
┃ 27 packets in total, first at 2021-03-02 21:26:13
┃ 192.168.1.73:51704 ->  144.126.208.148:22
┃ Client Proto : SSH-2.0-OpenSSH_8.4
┃ hassh        : b50f8371fb780ca7060e53c0b2cf6172
┃ Server Proto : SSH-2.0-OpenSSH_8.4
┃ hasshServer  : 2307c390c7c9aba5b4c9519e72347f34
┃ Summary of findings:
┃        4 Forward SSH login/init events
┃        1 Forward keystroke related events
┃ Detailed Events:
┃     packet     time(s)   delta(s)   Direction Indicator      Bytes   Notes
┃   -----------------------------------------------------------------------
┃       0         0         0         packet0   packet0          21              
┃       6         0.153     0.153     forward   key offered     556              
┃       7         0.159     0.006     forward   key accepted     16    Delta suggests hostkey was already in known_hosts or ignored
┃       11        0.311     0.152     forward   login prompt     84              
┃       12        0.315     0.004     forward   login success   500    Delta suggests Certificate Auth, pwd to cert null or non interactive
┃       22        0.79      0.475     forward   agent fwding    544    !! -A option used. Client private key sharing via SSH Agent Forwarding
┃
┃ Plotting packets 0-27 size histogram to '././packet-strider-ssh ssh-traffic.pcap stream 0 - Data Movement.png'
┃ Plotting packets 0-27 Data Movement predictions to '././packet-strider-ssh ssh-traffic.pcap stream 0 - Data Movement.png'
┃ Plotting packets 0-27 keystroke timeline to '././packet-strider-ssh ssh-traffic.pcap stream 0 - Keystrokes.png'
┃
┗━━━━ End of Analysis for stream 0

{% endhighlight %}

As you can see from the output, packet 22 was seen as an indicator for ssh agent forwarding (-A argument). The script is able to generate a hassh 
fingerprint only after observing the kex handshake. 

```bash
Client Proto : SSH-2.0-OpenSSH_8.4
hassh        : b50f8371fb780ca7060e53c0b2cf6172
Server Proto : SSH-2.0-OpenSSH_8.4
hasshServer  : 2307c390c7c9aba5b4c9519e72347f34
```

The plots that are generated are a histogram of packet sizes. Keystrokes and movements prediction. 

![packetStrider histogram](/images/tunnels/stream-0-histogram.png)

![stream-0-keystrokes](/images/tunnels/stream-0-keystrokes.png)

![stream-0-movements](/images/tunnels/stream-0-movement.png)
 
Let's go ahead and re-run packetStrider on a capture invoked with the same parameters but started seconds(?) after the ssh connection was initiatied. This should
demonstrate the point I have been trying to hammer home this whole time. 


```bash
pcap: ssh-traffic-tunnel.pcap 

... Loading full pcap : ssh-traffic-tunnel.pcap
... Getting streams from pcap:
    ...found stream 0
    ...found stream 1
    ...found stream 2
    ...found stream 3
    ...found stream 4
    ...found stream 5
    ...found stream 6
    ...found stream 7
    ...found stream 8
    ...found stream 9
    ...found stream 10
    ...found stream 11
... Loading stream 0
... Finding meta
... Finding hassh elements
... Building size matrix

   ... Ordering the keystroke packets
   ... Building features with window size = 2, stride = 1
       ... Calculating first packet timestamp
       ... Calculating last packet timestamp
       ... Calculating Max Packet delta
       ... Striding through windows of size 2

┏━━━━ Reporting results for stream 0
┃
┃ Stream 0 of pcap 'ssh-traffic-tunnel.pcap'
┃ 15 packets in total, first at 2021-04-03 18:46:51
┃ 192.168.1.91:59950 ->  143.198.238.169:22
┃ Client Proto : not contained in pcap
┃ hassh        : not contained in pcap
┃ Server Proto : not contained in pcap
┃ hasshServer  : not contained in pcap
┃ Summary of findings:
┃ Detailed Events:
┃     packet     time(s)   delta(s)   Direction Indicator      Bytes   Notes
┃   -----------------------------------------------------------------------
┃       0         0         0         packet0   packet0          68              
┃
```

The gist to note here is of course 'not contained in pcap'. The capture was purposely started after invoking the ssh command and its clear its rendered useless at this point. 
It cannot inher anything if it cannot satisfy the necessary criteria required for said evaluation. As a further exercise, the tool was also released as a zeek script, which 
you can load into your site/local.zeek. I use Brim fairly often and it relies on suricata and zeek (along with zq) on the backend to produce the necessary logs for the ingested PCAPs. 
I have added packetStrider (and packetSrider ) into my zeek config (zeekrunner with brim) so I now have this functionality whenever I am hunting. Next time on "On the Wire" we will take a 
look at DNS tunneling and potential detection methods. 
---
References:     

- [Figure 1](https://unix.stackexchange.com/questions/46235/how-does-reverse-ssh-tunneling-work) https://unix.stackexchange.com/questions/46235/how-does-reverse-ssh-tunneling-work
- [Figure 2](https://www.isabekov.pro/reverse-ssh-tunnel/): https://www.isabeok.pro/reverse-ssh-tunnels
- [HASSH](https://github.io/salesforce/hassh/): https://github.com/salesforce/hassh
- [packetStrider](https://github.com/benjeems/packetStrider/): https://github.com/benjeems/packetStrider



