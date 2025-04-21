---
layout: post
title:  TCP/IP in OCaml 1- TUN/TAP and IP Packet Parsing
date:   2025-04-21 12:06:35 -0400
categories: ocaml networking systems
tags: [ocaml, tun, tap, networking, packet-parsing, systems-programming]
---

As I set out to implement TCP/IP from scratch in OCaml I encountered a very basic problem in setting up the codebase: how do I send TCP packets without interference from my own system's implementation (And how does the interference work exactly?). To solve this problem, I used an existing TUN/TAP OCaml library (Which is a wrapper over an ioctl system call, which has to be done in C), that provides you a virtual network interface over which you can send IP packets to and from a user space program, entirely bypassing the kernel. 

TUN/TAP is usually used for implementing VPN, but I found several other blog posts implementing TCP/IP that used TUN/TAP, and wanted to provide my own understanding of what is going on. 

## Why not raw sockets? 

> [CITATION NEEDED] - have someone double check or find citation for following

>[CITATION NEEDED] - man pages link

TCP is a protocol that provides reliability guarantees on top of an unreliable packet delivery protocol (IP). The initial idea I had for implementing TCP as a user space program was to use raw sockets. Raw sockets allow user space implementation of a protocol on top of IP i.e. you can send/receive IP packets and rely on the system's network layer routing capabilities: The issue with implementing TCP in this manner is as follows: The user space TCP would be running on your local network. Hence when you have an incoming TCP packet wrapped inside of an IP packet, it reaches its local destination on the network and gets dispatched to the TCP stack because of the protocol number. So sending isn't an issue, it's receiving that's the problem. Of course, you could be writing the kernel implementation yourself. I am trying to do it as a user space program for simplicity. 

## TUN/TAP

>[CITATION NEEDED] - tun/tap docs

Instead, we can create a virtual network interface. This is not exactly what the docs say, but this is how I think of creating a TUN device to send IP packets to a user space program: You create a TUN device by an ioctl system call on /dev/net/tun. You can then assign an IP address space to this TUN device, using an IP reserved for private networks 
>[Citation NEEDED] - RFC 1918 

Now, even though our TCP program is running locally, I find it helpful to think of the assigned address space to be "external" to the local network on my machine; we have an interface via which packets can be sent (reading and writing to the file descriptor created by the ioctl system call). It can be seen in analogy to existing network interfaces on your system that correspond to physical mediums. 

For example, if you send a packet over the wifi interface, you can imagine that it will be handled externally; similarly, if the kernel receives and sends a packet over the tun device, it gets routed to the user space program that will handle all packets in the assigned address space. Instead of an OS and TCP stack, we have our own program!

```ascii
Local Machine
+--------------------------------------------------------------------+
|                                                                    |
|  +--------------------------------------------------------------+  |
|  |                                                              |  |
|  |                        Network Stack                         |  |
|  |                                                              |  |
|  +--------------------------------------------------------------+  |
|         ^                      ^                      ^            | 
|         |                      |                      |            |
|         v                      v                      v            |
|  +----------------+     +----------------+     +----------------+  |
|  |                |     |                |     |                |  |
|  |    eth0        |     |    wlan0       |     |    tun0        |  |
|  |  (Ethernet)    |     |   (WiFi)       |     |  (Virtual)     |  |
|  +----------------+     +----------------+     +----------------+  |
|         |                      |                      |            |
|         v                      v                      v            |
|  +----------------+     +----------------+     +----------------+  |
|  |                |     |                |     |                |  |
|  |  Router/       |     |  WiFi          |     |  My Tcp        |  |
|  |  Switch        |     |  Network       |     |  Program**     |  |
|  |                |     |                |     |                |  |
|  +----------------+     +----------------+     +----------------+  |
|                                                                    |
+--------------------------------------------------------------------+
```