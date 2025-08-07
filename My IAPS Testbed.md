---
title: On Benchmarking and Monitoring
draft: false
tags:
---

My IAPS repository can be found on github

[Github Link](https://github.com/hema2601/Custom_Packet_Steering)

To properly test my scheme, it was necessary to run many repetitions of individual setups and vary setups as well. Therefore, I wrote a testing infrastructure that proved to be extremely helpful.
Due to this being developed on the fly with little prior planning, there certainly are parts that are messy.
Anybody interested in upgrade/altering my setup is more than welcome to do so.

This document aims to explain my testbed so that it can be hopefully useful to somebody else as well.

# The Basic Idea

The requirements I had for my testbed straightforward:

1. Automatically run multiple instances of my experiments
2. Record data from individual experiments in a structured manner for later analysis
3. Automatically aggregate data from different experiments into easy-to-parse meta files
4. Automatically visualize my data in a way that is compatible with my work setup (All the data being located on a remote server with no GUI)




/