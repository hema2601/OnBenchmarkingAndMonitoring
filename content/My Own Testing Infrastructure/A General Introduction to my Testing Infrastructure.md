---
title: A General Introduction to my Testing Infrastructure
draft: false
tags:
---

The repository for my testing infrastructure can be found on github:
[Github Link](https://github.com/hema2601/testing_infrastructure)


To properly test my research, it was necessary to run many repetitions of individual setups and vary setups as well. Therefore, I wrote a testing infrastructure that proved to be extremely helpful.
Due to this being developed on the fly with little prior planning, there certainly are parts that are messy.
Anybody interested in upgrade/altering my setup is more than welcome to do so.

This document aims to explain my testbed so that it can be hopefully useful to somebody else as well.

# The Basic Idea

The requirements I had for my testbed are straightforward:

1. Automatically run multiple instances of my experiments
2. Record data from individual experiments in a structured manner for later analysis
3. Automatically aggregate data from different experiments into easy-to-parse meta files
4. Automatically visualize my data in a way that is compatible with my work setup (All the data being located on a remote server with no GUI)


To achieve this, the infrastructure consists of multiple components.

The individual parts and their interactions can be seen in the figure below:
![[Pasted image 20250808114154.png]]


Lets break this down in plain English.

A developer writes a [[Setup Iterator]]. This will deal with the general system and application setup. Things like how big the packets are, how long the experiment runs, how many cores will be given etc. When all this setup has been defined, the [[Experiment Iterator]] is called. It runs the same experiment multiple times, so that the final data can be obtained over multiple identical runs, and it scales up the connection number at a user-defined rate to a user-defined point. Ever time when one of those experiments is dispatched, the [[Single Experiment Script]] is executed. This script takes all the setup information and actually applies it to the system and runs the application with the right parameters while also activating the data collection. Then, at the end of the experiment when all the raw data has been obtained, a [[Raw Data Converter]] is called to convert the raw data - which is usually just some numbers - into readable json-files. This little loop will be repeated by the [[Experiment Iterator]] until all individual experiments have run. Then, the [[Setup Iterator]] dispatches the [[Summarizer]], who takes all the accumulated data of the individual experiments and summarizes them into one `summaries` folder. Lastly, after everything is executed, the user has the opportunity to run a [[Web Server]] on their server, which when accessed can immediately visualize the data that was summarized.

# How to Read this Guide

I am not exactly good at documenting my work, so there is a good chance that some of these documents will be hard to read. I tend to write too much and I am also so familiar with this infrastructure that I might forget to explain some crucial aspects.
In any case, keep the source code near so that you can look up anything that is not clear to you. I did my best to clean it up and add some comments.

## Learning by Doing

If you are more of a practical person, you might just want to start running an experiment. For that refer to [[Setting up and Running any Experiments in Your Own Environment]] followed by [[Running and Using the Web Server]]. Hopefully, everything will run smoothly and you will get some results. There is a good chance though that you will have to deal with some dependency issues since I did not try to install this on an untouched system.

## Get Into the Details

The documents in the `components` section describe in varying levels of detail the individual parts of my infrastructure. Either start with the [[Setup Iterator]] and follow the flow of a single experiment-suite execution, or start with the [[Single Experiment Script]] (by far the most taxing document), which is the centre of this infrastructure, and understand everything from the bottom up.

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
2. [[Running and Using the Web Server]]

Adding your own components to the infrastructure:
1. [[Adding New Data Sources]]
2. [[Adding New Visualizations]]
3. [[Adding New Setup Iterator (aka Experiment Script)]]
4. [[Adding New Experimental Parameters]]

