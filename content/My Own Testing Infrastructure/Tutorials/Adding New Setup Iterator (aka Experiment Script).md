---
title: Adding New Experimental Scripts
draft: false
tags:
---
 When adding a new setup iterator, follow the structure of the example given in the [[Setup Iterator|Setup Iterator Explanation]]. 

A simple skeleton of a setup iterator looks like this:

```bash
#!/bin/bash
source my_lib.sh

intf=$1
rep=$2
conns=$3

current_path=/home/hema/testing_infrastructure/

exponential=0

# Create Meta-Experiment Directory
name="Meta-Experiment"
mkdir $current_path/data/$name

# General Setup Parameters that don't change
set_intf $intf

# -- [Other general Setup Here] --

#========================

# Experiment Setups
exp_name="Sub-Experiment"

# -- [Other Experiment-Specific Setup Here] --

run_exp $exp_name $rep $conns $exponential $name

# -- [More Sub-Experiments Here] --

#========================

# Summarize
summarize $current_path $rep $conns $exponential $name
```

Just adjust the names to your liking and adjust the experimental parameters using the helper functions from `my_lib.sh`.

If you don't know what setup options you have, refer to the experiment's [[Single Experiment Script#Argument List|argument list]], to see what parameters you can change. There is no explicit document on which `my_lib.sh` function to use for which argument, but their names are pretty self-explanatory.

If you want to add a new experiment parameter, there is a tutorial on that [[Adding New Experimental Parameters|here]].

# The One Thing You Need to Pay Attention to 

In a perfect infrastructure, a user should be able to write a new setup iterator without having to change anything in the other core-infrastructure components. 

Unfortunately, this is not the case here, because of a flaw in the [[Summarizer]].

The [[Summarizer]] will only summarize data files in sub-directory folders that it can generate from a list of experiment names. That means that if your `exp_name` value is not yet on the Summarizer's `bases` list, you will have to add it. It is simple though.

Let's assume you used the above skeleton exactly as it is. You would have to go to the `merger.py` file and change the `bases` list like this:
```python
bases = ["RSS", "RPS", "RFS", [...], "Sub-Experiment"]

```

# Making it executable

You want to execute your script, but if you just created it with a tool like vim, you'll have to turn it executable. Just run something like this:

```bash
sudo chmod a+x my_setup_iterator.sh
```

If you copied your setup iterator from an existing one, this step won't be necessary.