---
title: On Benchmarking and Monitoring
---
# Hi. This is Maike.

While doing my Master's degree, I spent a lot of time teaching myself the ins and outs of kernel development, kernel observability, and benchmarking automation.

This document (now website I guess...) aims to conserve that in written form so that new students in [my former lab](https://netsys.ajou.ac.kr/) (and the occasional lost soul from the internet) can share in my knowledge on these topics and can potentially learn how to work with my benchmarking infrastructure, so that all my efforts were not for nothing.


# What you can find here

---
**[[A General Introduction to my Testing Infrastructure|My Own Testing Infrastructure]]**

During my research, I spent a significant amount of time automating my experiments. While I did not do a perfect job at making it general, I do think it turned out general enough so that it might be beneficial to other people wanting to run experiments at scale (mainly iperf...).

---

# What I may or may not include at a later point

**The Netdevice Subsystem**

My speciality in the kernel has been the netdevice subsystem and I have spent countless hours reading its code and making sense of its mechanisms. I want to preserve some of that both at a high-level, and at a code-level for the ones that are also obsessed with getting to the bottom of things and not afraid of the occasional snippet of assembly ;)

---
**What are Kernel Modules and How to Use Them?**

Throughout my research, kernel modules became one of the handiest tools in my tool box. You can add kernel functionality through them without having to change the actual kernel, expose interesting data through them, and do some powerful debugging with them. 

---
**My Master Thesis - IAPS**

IAPS (short for Interrupt Avoidance Packet Steering) is a novel packet steering scheme I developed for my Master's Thesis. Here, I want to get into the nitty-gritty of the actual kernel implementation of it. The kind of stuff that would have not made it into the paper/thesis because its just too much technical detail.

---

Thank you and Enjoy~

![[Changelog]]