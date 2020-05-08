---
permalink: /
title: "EdgeVPN"
header:
  overlay_color: "#5e616c"
  #overlay_image: /assets/images/banner.jpg
---

### EdgeVPN is an open-source software for deploying scalable Virtual Private Networks (VPNs) across distributed edge resources for distributed, containerized applications

It groups distributed nodes into a logical Ethernet. EdgeVPN has built-in support for packet capture, encryption, tunneling, forwarding, and NAT traversal, EdgeVPN builds on standard protocols, flexible software-defined networking, and a scalable overlay network architecture

### Scalable and self-configuring
Designed based on principles used in scalable, ring-structured key/value stores and peer-to-peer overlays, there are no central VPN traffic bottlenecks, and the network scales and configures itself as nodes are added/removed

### Run existing software
EdgeVPN exposes virtual network interfaces and private IP addresses allowing existing and future Ethernet/IP based software stacks to run, unmodified. EdgeVPN can provide a virtual cluster of Docker containers across edge/cloud resources

### Deploy anywhere
EdgeVPN transparently handles edge devices behind NATs and firewalls, encrypting and tunneling traffic across the Internet.

**Structured topology:** 
EdgeVPN currently implements a structured overlay topology where nodes self-organize into a ring ordered by unique node IDs, and with randomly-assigned “long-distance” links, following the Symphony peer-to-peer system. This topology is such that the average distance between two nodes can scale as a log(N) function, where N is the number of nodes. Topology handling is modular, such that other topologies can be implemented.

**Encrypted links:**
EdgeVPN links are encrypted and authenticated with standard SSL-based transport-layer security. It supports UDP-based DTLS over NAT hole-punched tunnels.

**Easy grouping:**
EdgeVPN uses the standard XMPP protocol with short messages to discover and exchange connection information with peers. Membership in private EdgeVPN groups can be easily configured for networks small to large in an XMPP server, such as the open-source OpenFire and eJabberd systems.

**Layer-2 networks:**
EdgeVPN exposes a virtual Ethernet to its endpoints, and supports the ARP protocol, and unicast and multicast IP applications. 

**Programmable and extensible:**
The core packet-switching in EdgeVPN is performed by programmable, software-defined Open vSwitch virtual switches, and endpoint interfaces are exposed via a virtual tap device. EdgeVPN can be deployed on physical and virtual machines, and in Docker containers.

**Built on standards:**
EdgeVPN leverages standards for NAT traversal (STUN, TURN, ICE), transport-layer security (TLS, DTLS), software-defined networking (OpenFlow), and short messaging (XMPP), and reuses the WebRTC open-source framework 