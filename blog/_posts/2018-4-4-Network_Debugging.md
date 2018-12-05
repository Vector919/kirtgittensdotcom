---
layout: post
title: Debugging web applications with networking tools
---

#### Debugging web applications with networking tools

I feel like I'm constantly runnining into issues debugging web applications involving external services with outside code. When this happens, I find that having access to some networking tools to do some debugging can save a *lot* of time, compared to trying to figure out what's going on some other way.

Being able to see what's going on and potentially change it at a network level can give you a level of insight into your web application problems, beyond what you might get from traditional debugging tools.

In case anyone else isn't aware of some of these tools and how they can help you, I'll walk through some example's of how I was able to use these to solve some issues.

#### 1. [netcat (nc)](http://netcat.sourceforge.net/)
When debugging issues with any network request, I find that the thing I need most often, is to spin up a simple server and view the raw contents of the it's requests. While there's lots of different ways you can do this, I think the most straightforward  way that's available on most platforms is netcat

```bash
# Listen for TCP connections on port 8000
nc -k -l 8000
```

After running this, I\'m able to point services that I\'m debugging to this port and treat it as a sort of mock call, so I can see the full data of the request they would be making.

#### 2. tcpdump
I was looking into a bug a while back, involving an STMP relay potentially mangling the email headers being sent to it. Based on all of the downstream code that generated the emails, everything looked fine before it got sent to the relay, but the emails I got out of it on the other side were still incredibly messed up.

I didn't have easy access to the source of the relay software or it's configuration, but I *did* have access to the box it was running on, so to figure out what it was doing to the emails, I used tcpdump:


```bash
tcpdump -A -i eth0 'port 25'
```
tcpdump does exactly what you might expect, print out the raw network packets from a particular network interface, and it's available by default many linux distributions.

To break this down a bit:

`-A`: Prints out each network packet in ASCII (instead of hex or some other encoding)
`-i eth0` Looks for packets from network interface eth0 (which is the default on many linux systems). You can find all of the network interfaces available with `ifconfig`

`'port 25'` Filters for traffic on port 25 (the default port used by the SMTP protocol)

Once you have the output from this command, you can filter it further using the regular bash command line tools. In my case, I was looking for a specific SMTP header in the packets, which I was able to do with `grep`:

```bash
tcpdump -A -i eth0 'port 25' | grep "Content-Transfer-Encoding"
```

Here is the   same tcpdump command, but with grep in order filter and find the transfer encoding in the packets.

Fairly effective at getting the raw data going through any service you want, even if you can't inspect it's code!


#### 3. [Wireshark](https://www.wireshark.org/)

In situations where you have access to the network traffic you're trying to inspect on your local machine (and even when you don't sometimes), Wireshark provides a much nicer interface around inspecting your local network traffic.

When debugging an issue with an FTP server, I was able to use wireshark locally to figure out the exact error, in a situation where my Java library was just throwing a generic exception.

Wireshark gives you the ability to generically capture packets, and filter them by many different criteria.

There's probably an infinite amount of depth to the [options available in Wireshark](https://www.wireshark.org/#learnWS) but all I needed for grabbing the SFTP was start capturing packets on my Wi-Fi interface with this filter:

![alt text](<https://storage.googleapis.com/imperial-flow-2206/wireshark_example.png> "Logo Title Text 1")

Once you've found the packets related to your particular issue Wireshark gives you a great interface that allows you to easily inspect the contents of every packet to see what could have gone wrong at any point.

