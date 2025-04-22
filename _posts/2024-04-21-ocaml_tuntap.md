---
layout: post
title:  TCP/IP in OCaml - TUN/TAP and IP Packet Parsing
date:   2025-04-21 12:06:35 -0400
categories: ocaml networking systems
tags: [ocaml, tun, tap, networking, packet-parsing, systems-programming]
---



As I set out to implement TCP in OCaml, I ran up against a very simple problem: where should it go? There is an existing kernel space tcp, and I want to implement my own as a user space program. The solution to this is TUN/TAP. TUN/TAP allows you to implement a user space program that can read IP packets over a virtual network device.

> [note] This post is a very intuitive overview as someone learning this stuff for the first time. It can be seen as the mental hurdles I faced trying to learn what tun/tap was about, and how I filled them in. This was also inspired by two other blog posts that explained using tun/tap to write a custom TCP, so I wanted to provide my own explanation of its use - [Julia-Evans](https://jvns.ca/blog/2022/09/06/send-network-packets-python-tun-tap/), [stacey-tay](https://stace.dev/rc-05-tcp-in-rust/).


## Why not raw sockets? 

TCP is a protocol that provides guarantees on top of an unreliable packet delivery protocol (IP). The initial idea I had for implementing TCP as a user space program was to use raw sockets. Raw sockets allow user space implementation of a protocol on top of IP i.e. I can send/receive IP packets and rely on the system's network layer routing capabilities: The issue with implementing TCP in this manner is that my system's TCP stack would get the packet as well as my user space program. Then I have two programs implementing TCP, and chaos ensues.

## TUN/TAP

Instead, we can create a virtual network device. The [linux kernel tun/tap docs](https://www.kernel.org/doc/Documentation/networking/tuntap.txt) are a good source to understand why we need tun/tap. 

```
1. Description
  TUN/TAP provides packet reception and transmission for user space programs. 
  It can be seen as a simple Point-to-Point or Ethernet device, which,
  instead of receiving packets from physical media, receives them from 
  user space program and instead of sending packets via physical media 
  writes them to the user space program. 

  In order to use the driver a program has to open /dev/net/tun and issue a
  corresponding ioctl() to register a network device with the kernel. A network
  device will appear as tunXX or tapXX, depending on the options chosen. When
  the program closes the file descriptor, the network device and all
  corresponding routes will disappear.

  Depending on the type of device chosen the userspace program has to read/write
  IP packets (with tun) or ethernet frames (with tap). Which one is being used
  depends on the flags given with the ioctl().
```

I also found this paragraph very insightful:

```
3. How does Virtual network device actually work ? 
Virtual network device can be viewed as a simple Point-to-Point or
Ethernet device, which instead of receiving packets from a physical 
media, receives them from user space program and instead of sending 
packets via physical media sends them to the user space program. 
```


I can best explain what is going on with a diagram 

```ascii
Local Machine
+--------------------------------------------------------------------+
|                                                                    |
|  +--------------------------------------------------------------+  |
|  |                                                              |  |
|  |                 Kernel Network Stack                         |  |
|  |                                                              |  |
|  +--------------------------------------------------------------+  |
|         ^                      ^                      ^            | 
|         |                      |                      |            |
|         v                      v                      v            |
|  +----------------+     +----------------+     +----------------+  |
|  |                |     |                |     |                |  |
|  |    eth0        |     |    wlan0       |     |    tun0        |  |
|  |  (Ethernet)    |     |                |     |  (Virtual)     |  |
|  +----------------+     +----------------+     +----------------+  |
|         |                      |                      |            |
|         v                      v                      v            |
|  +----------------+     +----------------+     +----------------+  |
|  |                |     |                |     |                |  |
|  |  Ethernernet   |     |  WiFi          |     |  MyTcpProgram  |  |
|  |  hardware      |     |                |     |                |  |
|  |                |     |                |     |                |  |
|  +----------------+     +----------------+     +----------------+  |
|                                                                    |
+--------------------------------------------------------------------+
```

For the usual network interfaces, for example eth0, there is a physical media backing (ethernet for example). Packets are received and sent back over an ethernet device. We can think of the "physical media" of the tun device being our user space program. Instead of receiving packets from some hardware, it receives packets from our user space program. And going the other way around, when we write packets to the device, they are written to our program. 

## ocaml-tuntap

The way a TUN or TAP device is created is via an ioctl system call on /dev/net/tun. This requires calling C code since ioctl is not provided in OCaml. The ocaml-tuntap library implements this, and is part of the MirageOS project which I thought was a neat connection. You can take a look at the source code, and see some of the ioctl stubs:

```c
//turntap_stubs.c
...
...
  if (group != -1) {
    if(ioctl(fd, TUNSETGROUP, group) < 0)
      tun_raise_error("TUNSETGROUP", fd);
...
...
```
Upon successful return you get a file descriptor corresponding to the virtual network device. Now, if you read from the file descriptor you get an entire IP packet (you do not have to repeatedly call Unix.read to read the right amount of bytes).  

## tun.ml

We will create our own tun module, which is a wrapper for the file descriptor returned by calling the ocaml-tuntap library. 

```ocaml
type t = {
  fd : Unix.file_descr;
  name : string;
}

let create name =
  let fd, actual_name = Tuntap.opentun ~pi:false ~devname:name () in
  Unix.clear_nonblock fd;
  { fd; name = actual_name }

let close t = Unix.close t.fd

let read t buf = Unix.read t.fd buf 0 (Bytes.length buf)

let write t buf =
  let len = Unix.write t.fd buf 0 (Bytes.length buf) in
  len 
```

## main.ml

The main driver assigns a private IP to the created tun device, and calls ip link to set it up. Then a buffer of 2000 bytes is used to read IP packets and parse them. IP packets could be larger, so that corner case is not currently handled. 

```ocaml
let hexdump buf size =
  let ihl = (Char.code (Bytes.get buf 0) land 0x0f) * 4 in  (* IHL is in 32-bit words, multiply by 4 for bytes *)
  Printf.printf "\nRaw IP packet bytes:\n";
  Printf.printf "Header (IHL: %d bytes):\n" ihl;
  for i = 0 to min (ihl - 1) (size - 1) do  (* Show header based on IHL *)
    Printf.printf "%02x " (Char.code (Bytes.get buf i));
    if (i + 1) mod 4 = 0 then Printf.printf "\n"
  done;
  if size > ihl then (
    Printf.printf "\nPayload:\n";
    for i = ihl to size - 1 do
      Printf.printf "%02x " (Char.code (Bytes.get buf i));
      if (i + 1) mod 4 = 0 then Printf.printf "\n"
    done
  );
  Printf.printf "\n%!"

let should_process_packet buf =
  let version = (Char.code (Bytes.get buf 0) lsr 4) in  (* Version is in high nibble of first byte *)
  let protocol = Char.code (Bytes.get buf 9) in  (* Protocol is at offset 9 in IP header *)
  version = 4 && protocol <> 0x11  (* IPv4 and not UDP *)

let run_command cmd =
  Printf.printf "Running command: %s\n" cmd;
  match Unix.system cmd with
  | Unix.WEXITED 0 -> ()
  | _ -> failwith ("Command failed: " ^ cmd)

let main () =
  let tun = Tun.create "tun0" in
  
  (* Configure IP and bring interface up *)
  run_command "ip addr add 10.1.2.1/24 dev tun0";
  run_command "ip link set tun0 up";
  Printf.printf "Configured tun0 with IP 10.1.2.1/24 and brought interface to operational state\n%!";
  
  let buf = Bytes.create 2000 in
  while true do 
    flush stdout;
    let packet_size = Tun.read tun buf in
    let packet_bytes = Bytes.sub buf 0 packet_size in
    if should_process_packet packet_bytes then (
      let ip_packet = Ip.parse_ip_packet packet_bytes in
      Printf.printf "\n=== Parsed IP Packet ===\n";
      Ip.pp_ip_packet (Format.formatter_of_out_channel stdout) ip_packet;
      Printf.printf "\n=== Raw Packet Hexdump ===\n";
      hexdump buf packet_size;
      flush stdout
    )
  done

let () = main () 
```

## ip.ml

And here is the initial pass at parsing and representing IP packets, according to RFC 791.

```ocaml
type version = IPv4 | IPv6

type protocol = 
  | ICMP
  | IGMP
  | TCP
  | UDP
  | Other of int

type t = {
  version : version;
  ihl : int;
  total_length : int;
  source_addr : int32;
  dest_addr : int32;
  payload : Bytes.t;
  protocol : protocol;
}

let ipv4_to_string addr =
  let b0 = Int32.to_int (Int32.shift_right_logical addr 24) in
  let b1 = Int32.to_int (Int32.shift_right_logical (Int32.logand addr 0x00FF0000l) 16) in
  let b2 = Int32.to_int (Int32.shift_right_logical (Int32.logand addr 0x0000FF00l) 8) in
  let b3 = Int32.to_int (Int32.logand addr 0x000000FFl) in
  Printf.sprintf "%d.%d.%d.%d" b0 b1 b2 b3

let string_to_ipv4 s =
  let parts = String.split_on_char '.' s in
  match List.map int_of_string parts with
  | [b0; b1; b2; b3] ->
      let addr = Int32.logor
        (Int32.shift_left (Int32.of_int b0) 24)
        (Int32.logor
          (Int32.shift_left (Int32.of_int b1) 16)
          (Int32.logor
            (Int32.shift_left (Int32.of_int b2) 8)
            (Int32.of_int b3)))
      in
      addr
  | _ -> failwith "Invalid IPv4 address format"

let parse_version buf = 
  match (Bytes.get buf 0 |> int_of_char) lsr 4 with
  | 4 -> IPv4
  | 6 -> IPv6
  | _ -> failwith "Invalid IP version"

let parse_ip_address buf start =
  Int32.logor
    (Int32.shift_left (Int32.of_int (Bytes.get buf start |> int_of_char)) 24)
    (Int32.logor
      (Int32.shift_left (Int32.of_int (Bytes.get buf (start + 1) |> int_of_char)) 16)
      (Int32.logor
        (Int32.shift_left (Int32.of_int (Bytes.get buf (start + 2) |> int_of_char)) 8)
        (Int32.of_int (Bytes.get buf (start + 3) |> int_of_char))))

let parse_ip_packet buf = 
  let version = parse_version buf in
  let ihl = (Bytes.get buf 0 |> int_of_char) land 0x0f in
  let total_length = Bytes.length buf in 
  let source_addr = parse_ip_address buf 12 in
  let dest_addr = parse_ip_address buf 16 in
  let payload = Bytes.sub buf (ihl * 4) (total_length - ihl * 4) in
  let protocol = match Bytes.get buf 9 |> int_of_char with
    | 1 -> ICMP
    | 2 -> IGMP
    | 6 -> TCP
    | 17 -> UDP
    | _ -> Other (Bytes.get buf 9 |> int_of_char)
  in
  { 
    version; 
    ihl; 
    total_length; 
    source_addr;
    dest_addr;
    payload;
    protocol;
  }

let pp_ip_packet fmt packet =
  let version_str = match packet.version with IPv4 -> "IPv4" | IPv6 -> "IPv6" in
  Format.fprintf fmt "IP Packet:@\n";
  Format.fprintf fmt "  Version: %s@\n" version_str;
  Format.fprintf fmt "  IHL: %d (words)@\n" packet.ihl;
  Format.fprintf fmt "  Total Length: %d bytes@\n" packet.total_length;
  Format.fprintf fmt "  Source: %s@\n" (ipv4_to_string packet.source_addr);
  Format.fprintf fmt "  Destination: %s@\n" (ipv4_to_string packet.dest_addr);
  Format.fprintf fmt "  Payload Length: %d bytes@\n" (Bytes.length packet.payload);
  Format.fprintf fmt "  Protocol: %s@\n" (match packet.protocol with
    | ICMP -> "ICMP"
    | TCP -> "TCP"
    | UDP -> "UDP"
    | IGMP -> "IGMP"
    | Other p -> Printf.sprintf "Other (%d)" p)

let show_ip_packet packet =
  Format.asprintf "%a" pp_ip_packet packet
```

Once it is up and running, I can ping an IP that's in the address range of the tun device, and see the output: 

```
=== Parsed IP Packet ===
IP Packet:
  Version: IPv4
  IHL: 5 (words)
  Total Length: 84 bytes
  Source: 10.0.0.1
  Destination: 10.0.0.3
  Payload Length: 64 bytes
  Protocol: ICMP

=== Raw Packet Hexdump ===

Raw IP packet bytes:
Header (IHL: 20 bytes):
45 00 00 54 
5f 13 40 00 
40 01 c7 92 
0a 00 00 01 
0a 00 00 03 
Payload
...
...
```
Now there is basic infrastructure set up to go about implementing TCP as a user space program! The full code can be found on my [github](https://github.com/artjomPlaunov/my_tcp).

## Sources
- [Raw Sockets](https://man7.org/linux/man-pages/man7/raw.7.html)
- [TUN/TAP Docs](https://docs.kernel.org/networking/tuntap.html)
- [RFC 1918 - Private Internets](https://datatracker.ietf.org/doc/html/rfc1918)
- [RFC 791 - IP](https://datatracker.ietf.org/doc/html/rfc1918)
- [Ocaml tun-tap source](https://github.com/mirage/ocaml-tuntap/blob/main/lib/tuntap_stubs.c)


