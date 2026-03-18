---
title: "The Tale of 2 Henrys and BMP"
date: 2025-03-06
speakerType: presentation
location: "DKNOG15"
locationUrl: "https://events.dknog.dk/event/32/"
image: "/speaker/tale-of-two-henrys-and-bmp/images/dknog.png"
links:
  - name: Video
    url: "https://youtu.be/e3OhlqFvwJg?si=JGmz8ATHulvWx-f5"
  - name: Slides
    url: "https://drive.google.com/file/d/1FrExg3i-Ieh9vLlQq4HXYCXLpS0W7Vzc/view"
description: "What building cars can teach us about building software and BMP — lessons from Henry Royce and Henry Ford applied to network automation."
tags: [
  "BGP",
  "BMP",
  "network automation",
  "software development",
]
---

*What building cars can teach us about building software and BMP*

In the late 19th century, two industrial titans were born within a few months of each other but an ocean apart. Both of these men, Henry Royce and Henry Ford, were obsessed with precision engineering and, fortunately for many of us, cars. They focused on building the best motor cars possible, though they achieved their goals in very different ways: Royce was driven by perfection, Ford by production.

[Michael Daly](https://www.linkedin.com/in/michaeladaly/) (Senior Director of Engineering) and [Bart Dorlandt](https://www.linkedin.com/in/bartdorlandt/) (Senior Network Automation Engineer) are working at Imperva where they are undergoing a complete rewrite of the Automation platform and we are using some of the ideas that these engineers have taught us.

This presentation will explore our vision for building better, more sustainable tools and discuss how the Network Automation team at Imperva is implementing these principles in our workflow.

From the technical side we zoom in on a recent project using BGP Monitoring Protocol (BMP). Where we previously had manual network verification, which improved to screen scraping, now having a push model from the router to a central database. This gives us an "offline" state and allows the customer to self verify their configured and advertised prefixes are accepted and learned as expected. This didn't happen without challenges. We will share the pitfalls and the challenges and how we faced and overcame them.
