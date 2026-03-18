---
title: "Deploying the BGP Monitoring Protocol (BMP) at ISP Scale"
date: 2024-11-13
speakerType: podcast
location: "Packet Pushers - Heavy Networking"
locationUrl: "https://packetpushers.net/podcasts/heavy-networking/hn-759-deploying-the-bgp-monitoring-protocol-bmp-at-isp-scale/"
image: "/speaker/hn759-deploying-bmp-at-isp-scale/images/hn759-artwork.png"
links:
  - name: Episode page
    url: "https://packetpushers.net/podcasts/heavy-networking/hn-759-deploying-the-bgp-monitoring-protocol-bmp-at-isp-scale/"
  - name: Apple Podcasts
    url: "https://podcasts.apple.com/us/podcast/heavy-networking/id370842767"
  - name: Spotify
    url: "https://open.spotify.com/show/7GlOoc33YmMT9j9hrvKH0y"
  - name: Overcast
    url: "https://overcast.fm/itunes370842767"
  - name: PocketCasts
    url: "https://pca.st/XOMu"
description: "How Bart designed and built a BMP solution at ISP scale — architecture, challenges with millions of records, and why Kafka was essential."
tags: [
  "BGP",
  "BMP",
  "network monitoring",
  "ISP",
  "Kafka",
  "network automation"
]
---

The BGP Monitoring Protocol, or BMP, is an IETF standard. With BMP you can send BGP prefixes and updates from a router to a collector before any policy filters are applied. Once collected, you can analyze this routing data without any impact on the router itself. On today's Heavy Networking, we talk with Bart Dorlandt, a network automation solutions architect. An ISP approached Bart with a use case for BMP, and he designed and built a solution to serve the ISP's customers.

We discuss what BMP is good for, how it works, and why Bart needed to use BMP. We also get into the tools he used to build his solution, the architecture he designed, the challenges he ran into in dealing with millions of records, why Kafka was essential for scaling, and more.
