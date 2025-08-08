---
title: Setup Iterator
draft: false
tags:
---
<<[[Experiment Iterator|previous]]|[[Summarizer|next]]>>

The setup iterator is the starting point of any experiment suite.
It exists to set up the correct environment before starting the actual experiment and coordinating multiple setups. 

The setup iterator is not a core part of the infrastructure. Rather, after the entire infrastructure is setup, the developer can write a new setup iterator every time when they come up with a new type of experiment (if the infrastructure is written properly ㅜㅜ).

In terms of the github repository, the multiple setup iterators are collected in `./experiment`

Lets take one of the simpler setup iterators: `baseline_experiment.sh` (Slightly modified for clarity)
This is an experiment suite for running all baseline experiments for preexisting packet steering schemes.
It was necessary because, while IAPS was constantly changing, these implementations weren't so it was necessary to isolate the IAPS test suite, which had to be run often, from the baseline data, which only really needed to be run once.

```BASH
#!/bin/bash

#========= Initialization
source my_lib.sh

intf=$1
rep=$2
conns=$3

current_path=/home/hema/Custom_Packet_Steering

exponential=0

name="Main_Baseline"

#========= General Setup
set_intf $intf
set_sep $ON
set_gro $ON
set_mss 1460
set_core_start 0
set_core_num 8

set_latency_measures $ON
set_pkt_size_measures $ON

#========= 1st Exp Setup
exp_name="RSS"

set_rss $ON
set_queue 4

run_exp $exp_name $rep $conns $exponential
#========= 2nd Exp Setup
exp_name="RPS"

set_rps $ON
set_queue 1

run_exp $exp_name $rep $conns $exponential

#========= 3rd Exp Setup
exp_name="RFS"

set_rfs $ON

run_exp $exp_name $rep $conns $exponential

#========= Wrap Up
rm -r $current_path/summaries

python3 $current_path/merger.py $2 $3 $exponential

mkdir $current_path/data/$name

mv $current_path/data/R* $current_path/data/$name
mv $current_path/data/IAPS* $current_path/data/$name
mv $current_path/summaries $current_path/data/$name
```

Lets go through this step by step:

## Initialization

```BASH
#========= Initialization
source my_lib.sh

intf=$1
rep=$2
conns=$3

current_path=/home/hema/Custom_Packet_Steering

exponential=0

name="Main_Baseline"
```

First, we include our library. This is a file that contains my own implementation of common functions used in these setup iterators. This was done so that I don't have to write the same functions over and over again and potentially risking the files go out of sync.

Then, we get our three arguments: The name of the interface, the number of repetitions for each individual experiment, and the number of connections each experiment should be run for.

These are passed as arguments rather than in-code variables, because these are the ones that are most frequently altered.

The `exponential` variable decides whether the experiment's connection number should be incremented exponentially or not. 
With `conns=5` and `exponential=0`, the experiments run for 1, 2, 3, 4 and 5 connections.
With `conns=5` and `exponential=1`, the experiments run for 1, 2, 4, 8 and 16 connections.
This should probably be a command line argument as well.

Lastly, the name of the experiment suite. This is the name of the folder that will be created in the `./data` directory under which all the individual experiments and the final summary will be saved. This is also the name by which the data will later be accessed from the webserver.

## General Setup

```BASH
#========= General Setup
set_intf $intf
set_sep $ON
set_gro $ON
set_mss 1460
set_core_start 0
set_core_num 8

set_latency_measures $ON
set_pkt_size_measures $ON
```

In the general setup, we define the 
## Setup 1 

```BASH
#========= 1st Setup
exp_name="RSS"

set_rss $ON
set_latency_measures $ON
set_pkt_size_measures $ON

run_exp $exp_name $rep $conns $exponential
```

For each experiment, we need to give the experiment a name and define the things that are unique to it.
Here, we want to test RSS, so that is the name we give the experiment 

<<[[Experiment Iterator|previous]]|[[Summarizer|next]]>>