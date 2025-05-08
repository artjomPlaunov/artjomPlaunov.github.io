---
layout: post
title:  TCP/IP in OCaml Interlude - Connecting two TUNS
date:   2025-05-02 12:06:35 -0400
categories: ocaml networking systems
tags: [ocaml, tun, tap, networking, packet-parsing, systems-programming]
---

In this interlude post I go over some local network setup to allow forwarding TCP packets between two tun devices. This is a simple local network set up that allows us to test out having two separate user space programs running the custom TCP stack and establishing a connection with one another. The point with this was to have the packets actually hitting my kernel's network stack, even in this toy implementation. Later on, I can change the local network setup if I want to have the packets be forwarded to the outside world, but I wouldn't have to change my TUN-based TCP implementation. 

This is what worked for me on my fedora machine. I hope to backfill this section with some knowledge from TCP/IP Illustrated Volume 1, especially the sections on IP protocol, firewalls, and forwarding. 

Here is my current Makefile. Note that the actual tun device creation and setting the link state to UP happens in the OCaml code when the TCP server spins up. In the next section, we will see some packet forwarding in action. The main things here are setting the ip_forward flag to 1, and also adding the tun interfaces to a trusted zone for my firewall. Also, for sending TCP packets I had to add a prerouting rule for the tun devices using iptables. 

In the next section, we will see the packet forwarding in action. 

```make
.PHONY: all build clean run

all: build

preroute: 
	iptables -t raw -I PREROUTING -i tun0 -p tcp -j NOTRACK
	iptables -t raw -I PREROUTING -i tun1 -p tcp -j NOTRACK

clean_preroute:
	iptables -t raw -D PREROUTING 1 || true
	iptables -t raw -D PREROUTING 1 || true

forwarding: 
	sysctl -w net.ipv4.ip_forward=1
	firewall-cmd --zone=trusted --add-interface=tun0
	firewall-cmd --zone=trusted --add-interface=tun1
	firewall-cmd --permanent --zone=trusted --add-interface=tun0
	firewall-cmd --permanent --zone=trusted --add-interface=tun1
	firewall-cmd --reload

clean_forwarding:
	sysctl -w net.ipv4.ip_forward=0
	firewall-cmd --zone=trusted --remove-interface=tun0 || true
	firewall-cmd --zone=trusted --remove-interface=tun1 || true
	firewall-cmd --permanent --zone=trusted --remove-interface=tun0 || true
	firewall-cmd --permanent --zone=trusted --remove-interface=tun1 || true
	firewall-cmd --reload

network:
	$(MAKE) clean_preroute
	$(MAKE) preroute
	$(MAKE) forwarding

clean_network:
	$(MAKE) clean_preroute
	$(MAKE) clean_forwarding

build:
	sudo $(MAKE) network
	sudo env PATH==$$PATH dune build

clean:
	sudo $(MAKE) clean_network
	sudo env PATH==$$PATH dune clean

view_network:
	ip link
	ip route
	sudo iptables -t raw -L -v -n --line-numbers

run-host:
	sudo env PATH=$$PATH dune exec ./host.exe tun0 10.0.0.1/24 10.0.1.5

trace-host: 
	sudo env PATH=$$PATH eio-trace run -- ./_build/default/host.exe tun0 10.0.0.1/24 10.0.1.5

run-peer:
	sudo env PATH=$$PATH dune exec ./peer.exe tun1 10.0.1.1/24 10.0.0.7

```
