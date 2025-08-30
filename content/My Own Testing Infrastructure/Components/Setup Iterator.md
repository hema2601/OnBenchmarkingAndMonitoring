---
title: Setup Iterator
draft: false
tags:
---


The setup iterator is the starting point of any experiment suite.
It exists to set up the correct environment before starting the actual experiment and coordinating multiple setups. 

The setup iterator is not a core part of the infrastructure. Rather, after the entire infrastructure is setup, the developer can write a new setup iterator every time when they come up with a new type of experiment.

An example setup operator can be found in the `experiment/` folder
Lets take a look at it: `baseline_experiment.sh`
This is an experiment suite for running all baseline experiments for the Linux-native packet steering schemes (except aRFS).

```bash
#!/bin/bash
source my_lib.sh

intf=$1
rep=$2
conns=$3

current_path=/home/hema/testing_infrastructure/

exponential=0

# Create Meta-Experiment Directory
name="Main_Baseline"
mkdir $current_path/data/$name

# General Setup Parameters that don't change
set_intf $intf
set_sep $ON
set_gro $ON
set_mss 1460
set_core_start 0
set_core_num 8

# Experiment Setup 1: RSS
exp_name="RSS"

set_rss $ON
set_queues 4

run_exp $exp_name $rep $conns $exponential $name

# Experiment Setup 2: RPS
exp_name="RPS"
set_queues 1

set_rps $ON

run_exp $exp_name $rep $conns $exponential $name

#Experiment Setup 3: RFS
exp_name="RFS"

set_rfs $ON

run_exp $exp_name $rep $conns $exponential $name


# Summarize
summarize $current_path $rep $conns $exponential $name
```

Lets go through this step by step:

## Initialization

```bash
source my_lib.sh

intf=$1
rep=$2
conns=$3

current_path=/home/hema/testing_infrastructure/

exponential=0

# Create Meta-Experiment Directory
name="Main_Baseline"
mkdir $current_path/data/$name
```

First, we include our library. This is a file that contains my own implementation of common functions used in these setup iterators. This was done so that I don't have to write the same functions over and over again and potentially risking the files to go out of sync.

Then, we get our three arguments: The name of the interface, the number of repetitions for each individual experiment, and the number of connections each experiment should be run for.

These are passed as arguments rather than in-code variables, because these are the ones that are most frequently altered.

The `exponential` variable decides whether the experiment's connection number should be incremented exponentially or not. 
With `conns=5` and `exponential=0`, the experiments run for 1, 2, 3, 4 and 5 connections.
With `conns=5` and `exponential=1`, the experiments run for 1, 2, 4, 8 and 16 connections.
This should probably be a command line argument as well.

Lastly, the name of the experiment suite. This is the name of the folder that is created in the `data/` directory under which all the individual experiments and the final summary will be saved. Those directories will be referred to as "meta-experiment" directories from now on. This is also the name by which the data will later be accessed from the [[Web Server|web server]].

## General Setup

```bash
# General Setup Parameters that don't change
set_intf $intf
set_sep $ON
set_gro $ON
set_mss 1460
set_core_start 0
set_core_num 8
```

In the general setup, we define the values of our experiment that don't change. In this case, we use the same network interface, we always want processing tasks to be isolated from each other, we always use GRO, keep a large packet size, and use the cores 0 to 7.

An explanation to all the different arguments that can be changed in the experiment, you can refer to the [[Single Experiment Script#Argument List|argument list]] of the [[Single Experiment Script|main experiment script]].
## Sub-Experiment Setups 

```bash
# Experiment Setup 1: RSS
exp_name="RSS"
set_rss $ON
set_queues 4
run_exp $exp_name $rep $conns $exponential $name

# Experiment Setup 2: RPS
exp_name="RPS"
set_queues 1
set_rps $ON
run_exp $exp_name $rep $conns $exponential $name

#Experiment Setup 3: RFS
exp_name="RFS"
set_rfs $ON
run_exp $exp_name $rep $conns $exponential $name
```

For each experiment, we need to give the experiment a name and define the things that are unique to it. The `exp_name` here is what will be referred to as the "sub-experiment" name. This is the name of one experiment that is repeated multiple times for different connection values, and it coexists with other sub-experiments under the bigger meta-experiment umbrella.

For our RSS experiment, we want to turn on RSS and we give it 4 RX queues, so that among the 8 cores we use for the experiment, 4 will be doing network processing and 4 will be doing application processing. Then, we call the [[Experiment Iterator]], that will take care of dispatching the individual iperf experiments.

For RPS and RFS, we similarly set a new packet steering scheme and adjust the number of RX queues. Note that the number of RX queues is not set again for RFS. This is because RPS already set the number to 1 and since for our RFS experiment, we also just need 1 RX queue, we do not need to explicitly reset it.

## Summarizing the Data

```bash
# Summarize
summarize $current_path $rep $conns $exponential $name
```

After all the sub-experiments of our meta-experiment have concluded, we want to summarize the data so it can be easily visualized. The `summarize` function is also defined in `my_lib.sh` and dispatches the [[Summarizer]].


# my_lib.sh

`my_lib.sh` is not big enough to warrant its own file, but it should still be explained a little bit.
As stated before, this is a library that is provided to the Setup Iterators so that their code can be clean and easily updateable. The code within it is very straight-forward, so you should be able to understand the logic by looking at it.

The one thing I wanted to point out about `my_lib.sh` is its `ARG_STRING` variable.
`ARG_STRING` is the string that is passed to the [[Experiment Wrapper]] to actually execute the experiment. It is append-only. Every time when the Setup Iterator uses another function to set an argument, it will be appended to `ARG_STRING`.

Earlier, we saw that we kept changing the packet steering scheme using functions like `set_rss`, `set_rps`, and `set_rfs`. These arguments are continuously appended, so the `ARG_STRING` will look something like this:

```
-P rss -P rps -P rfs
```

This is okay, because the [[Experiment Wrapper]] will only consider the *last* value given for any option. 