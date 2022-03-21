---
title: "Swim protocol"
date: 2022-03-12T16:53:46-03:00
categories: ["today-i-learned", "distributed", "protocols", "papers"]
draft: true
---

# Intro

I have just joined a company that uses [Consul](https://www.consul.io/) among several other tools. Consul is usually used for service discovery and while reading its documentation, i found out it uses [SWIM](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf) -- a [gossip protocol](https://en.wikipedia.org/wiki/Gossip_protocol) -- to [manage cluster membership](https://www.consul.io/docs/architecture/gossip).

# References

[SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol](https://www.cs.cornell.edu/projects/Quicksilver/public_pdfs/SWIM.pdf)  
[Consul gossip architecture](https://www.consul.io/docs/architecture/gossip)  
[Gossip protocol - wikipedia](https://en.wikipedia.org/wiki/Gossip_protocol)
