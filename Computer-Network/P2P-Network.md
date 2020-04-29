# P2P (Peer-to-Peer) Network
---
## Motivation
The goal of a P2P network is to offload the burden on the service provider by transfering computational, storage bandwidth resource responsibilities to the clients.
It is a service architecture used to provide highly scalable & cost efficient services.

---
## Application
A P2P network can be used for:
* __Content Distribution:__ Napster, BitTorrent
* __Internet Telephony:__ Skype
* __Video Streaming:__ WebRTC
* __Computation:__ Bitcoin

---
## Architecture
### Centralized Lookup Service
If I have some content that I want to share, how do I let everybody else on the network know that it exists? 
Or if I want to search content on the network, how do I know who has it? 

A centralized lookup service is a directory that keeps tracks of who has what content and how it can be contacted.

![Centralized Lookup Service](images/Centralized-Lookup-Service.png)

---
## Challenges
* __Dynamicity:__ Each node on the network can be online & offline at any time. How to preserve continuity of service?
* __Heterogeneity:__ Each node on the network has different capacity (bandwidth, computation power). How to preserve reliability?
