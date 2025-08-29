---
title: A General Introduction to my Testing Infrastructure
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


To achieve this, the infrastructure consists of multiple components.

The individual parts and their interactions can be seen in the figure below:
![[Pasted image 20250808114154.png]]


# How to Read this Guide

I am not exactly good at documenting my work, so there is a good chance that some of these documents will be hard to read. 

# What Can you Learn Here?

Learn about the individual components:
1. [[Single Experiment Script]]
2. [[Experiment Wrapper]]
3. [[Raw Data Converter]]
4. [[Experiment Iterator]]
5. [[Setup Iterator]]
6. [[Summarizer]]
7. [[Web Server]]

Learn about configuring the infrastructure for your own system:
1. [[Setting up and Running any Experiments in Your Own Environment]]

Adding your own components to the infrastructure:
1. [[Adding New Data Sources]]
2. [[Adding New Visualizations]]
3. [[Adding New Experimental Scripts]]
4. [[Adding New Experimental Parameters]]

