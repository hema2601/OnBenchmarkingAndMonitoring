---
title: What could have been...
draft: true
tags:
---
 

This testing infrastructure is imperfect. I always dreamed of taking some time and redesigning it so that it would be more generally applicable, but alas, my time is over ㅜㅜ

Here, I intend to write down the major flaws with this system and what I would have wanted to change (and sometimes what my plan was) if I had had sufficient time.
Maybe somebody will carry on my legacy~

# 1. Naming, Naming, Naming

The naming in this was an absolute NIGHTMARE.

As a refresher, the naming structure of the data is as such:

## Experiment Suite Naming

Under the `./data` folder are the experiment suites. Those have no structure and are user-defined. This is a problem for multiple reasons:

1. Running the same experiment suite twice will either overwrite or fail (because a directory with the same name already exists)
2. Humans are flawed and the names that I thought were strong enough to at the time for me to understand my intentions were **absolutely not**

## Experiment Naming



# 2. Lack of Metadata

This one is HUGE. 
Keeping track of the metadata of your experiments is incredibly helpful for detecting erroneous setups early on and adding context to experiments you ran ages ago.

I had two goals for my infrastructure:

1. Write a file containing the metadata as defined in the setup (queue nums, core layout, module parameters etc.)
2. Write functions that verify that the intended setup was actually performed  (if specifying RPS usage, run an independent query that checks whether the system is actually set up properly)

The first one would be easy - almost trivial - after the introduction of the [[Setup Iterator]]. It never happened because the setup iterator actually was a very recent introduction to the testing infrastructure.

The second one is harder, even impossible for some metrics.

In a perfect world, each possible setup option would be implemented like this:

```Python
class setup_option:
	def __init__(self, some_setup_data, prio):
		self.setup = some_setup_data
		self.prio = prio
			
	def perform_setup():
		# how to actually configure the system 

	def confirm_setup():
		# get system config from system
		curr_setup = ...

		if curr_setup != self.setup:
			if self.prio is CRITICAL:
				abort()
			if self.prio is WARNING:
				warn()

```

The `perform_setup` method would be called in the Setup Iterator, and the `confirm_setup` method would be called before experiment execution. It would have been beautiful.

However, this is not possible for a lot of setup parameters.

While for example a module parameter could easily be implemented like this:
```Python
def perform_setup():
	echo self.value > /path/to/module/parameter

def confirm_setup():

	val = read(/path/to/module/parameter)
	if val != self.value:
		... 
```

Setup parameters such as CPU assignments (values used for taskset) or the number of connections used in iperf are not that easily defined as simple function calls.

Anyways, you get the idea.

The ultimate goal was to have a system that automatically matches the intended setup to the actual setup and reports any errors. This is incredibly difficult though.

So if you want to change anything, *at least* write the setup parameters into some metatdata json that can be manually checked or integrated into the webserver to be able to check immediately what setup the displayed data was generated from. 

# 3. Proper Logging

If you ran any of my experiments, you know the output is a *mess*.
Totally irrelevant info 

# 4. The Monster that is the experiment script 

(separate the experiment script into separate system setup, experiment output, and experiement scripts to enable further generalization)

# 5. The Webserver Interface