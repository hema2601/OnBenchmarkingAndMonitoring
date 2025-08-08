---
title: Single Experiment Script
draft: false
tags:
---
<<|[[Experiment Wrapper|next]]>>

The single experiment script is the heart of the testing infrastructure.

This is because in the beginning, my 'testing infrastructure' only consisted of this very bloated piece of code. Eventually, I started to build the other components around it to make everything more manageable and  configurable. 

Still, most of the actual experiment setup is happening in this script. Everything else is just automation and output data processing, really.

Explaining this piece of code in a concise manner is a bit difficult since the code in it is not exactly clean.

Let's break this down into its three most fundamental functionalities:
1. Running the target application
2. Performing the system configurations
3. Orchestrating data collection

At the end of the document, I'll define all the arguments that the script takes and their purpose and implementation.

# Running the Target Application

The target application run by the experiment script is iperf3.

I always thought that one day, it would be neat if you could plug in different applications as well, but I never got to implement that.

The actual iperf execution is a one-liner, but let's break it down into its logical components: CPU core pinning, local server execution, and remote client execution.

## CPU core pinning

The range of CPU cores for the experiment is defined by the arguments [[#Beginning of CPU Core Range - *core_start*|core_start]] and [[#Number of Total CPU Cores - *core_num*|core_num]]. According to whether we want to run the application and networking [[#App and Network Isolation - *separate*|separate]] or not, another variable set `APP_CORE` and `APP_CORE_NUM` will be set. Those two variables describe the beginning of the application-dedicated CPUs and their total number.

Therefore, to contain our iperf application strictly within those dedicated cores, we use `taskset` before executing our iperf server. This looks like this: 
``` shell
taskset -c "$APP_CORE-$((APP_CORE + APP_CORE_NUM - 1))" # Rest of the command
```

That means that if we have 4 cores available to the application (`APP_CORE_NUM=4`) and those cores start from core number 1 (`APP_CORE=1`), we would run:
``` shell
taskset -c 1-4 # Rest of the command
```

Like this, iperf will only run on our assigned cores.

## Local Server Execution

This is the part of the command that runs iperf on the host that is executing the experiment script.

If you ever executed iperf, you know that most of the options are set on the client, not the server.

Hence, the server execution is not affected by any command-line arguments and is always the same:

``` shell
$IPERF_BIN -s -1 -J $IPERF_CUSTOM_ARGS > $current_path/iperf.json &
```

The `-s` option indicates that we are running the server, the `-1` options says that we want to end the server after one run instead of having it run persistently, and `-J` makes iperf print its output in json-format.
iperf's output is then redirected into an iperf.json file in the `current_path` and we use the `&` to detach the iperf process from hte command line, so we don't wait for it to finish.

The `IPERF_BIN` and `IPERF_CUSTOM_ARGS` serve the purpose of flexibly converting between running vanilla iperf or my [custom iperf](https://github.com/hema2601/iperf).

At the beginning of the script, those two are initialized to the normal iperf command and an empty string.
``` shell
IPERF_BIN=iperf3
IPERF_CUSTOM_ARGS=""
```

Then, the script checks whether my custom iperf is installed on the system, and only if it finds it, will it change the command and argument string.

``` shell
if command -v iperf3_napi &> /dev/null
then
	IPERF_BIN=iperf3_napi
	IPERF_CUSTOM_ARGS="--server-rx-timestamp 20,5000"
    echo "Using custom iperf3"
fi
```

This has the effect that running the experiment script on a host without my custom iperf will just run vanilla iperf, but on a host that has my custom iperf installed, it automatically uses the custom version.

> [!caution]- Naming of the custom iperf
> The binary of my custom iperf being called `iperf3_napi` is not something that works out-of-the-box when installing from my git repository. When installing custom iperf, I only compile it locally using `make` and created a symlink using `ln` in `/usr/bin` called `iperf3_napi` that point to the compiled binary in my local repository. Do not run `make install` on my custom iperf because it might overwrite the normal iperf and I have not tested my iperf extensively enough to recommend this.

The custom iperf arguments are hardcoded at the moment, but those might be another good candidate to be [[Adding New Experimental Parameters|added as a new parameter]]!

## Remote Client Execution

Now that we executed the server, the last thing left to do is to execute the client to kick-start the experiment. The problem is that the client needs to be run on a remote node, but we want the experiment to run automatically without us sshing onto another server.

This can be fixed with these two things: ssh remote command execution and password-less ssh.
The first one is a basic feature of `ssh` that I wasn't aware of. If you pass a string into your `ssh` command, it will execute that string as a command on the remote server. That means that if you run
``` shell
ssh username@ip "ls"
```
you'll just print your remote home-directory to the shell. Pretty neat.

Still, you'd have to put in your password, which would stall the experiment. So for this to fully work you have to set up password-less ssh between your nodes. How to do that is covered [[Setting up and Running any Experiments in Your Own Environment|here]].

Knowing that, we can now look at the remote client execution:
``` shell
ssh $remote_client_addr "iperf3 -c ${server_ip} -P ${conns} -M ${mss} -t ${time} > /dev/null"&
```

We see that the client relies on some command-line arguments, like [[#Maximum Segment Size - *mss*|mss]], [[#iperf Experiment Duration - *time*|time]], and [[#Connection Number - *conns*|conns]].
Apart from that, it requires the ip address of your NIC on the server (`server_ip`) and the way to ssh into you remote client (`remote_client_addr`).
Those are hardcoded at the beginning of the script and will have to be changed as part of the setup process when [[Setting up and Running any Experiments in Your Own Environment|first setting up the infrastructure]].

> [!caution]- Pay attention to IP addresses
> If you are running serious experiments, you will not be using your motherboard's native 1G interface but some high-bandwidth NIC. Note that the IP address used in `remote_client_addr` should be the remote client's *1G interface* and the IP address in `server_ip` should be your server's *high-bandwith IP*. This is so that ssh traffic won't interfere with your actual experiment's traffic.

Lastly, note that we are sending the iperf client's output into `/dev/null`. This will just throw it away, since to this point, I haven't had any use for the client-side output.

## Putting it all together

Now lets take a look at the full application execution:
``` shell
taskset -c "$APP_CORE-$((APP_CORE + APP_CORE_NUM - 1))" $IPERF_BIN -s -1 -J $IPERF_CUSTOM_ARGS > $current_path/iperf.json & ssh $remote_client_addr "iperf3 -c ${server_ip} -P ${conns} -M ${mss} -t ${time} > /dev/null"&
IPERF_PID=$!
[...]
tail --pid=$IPERF_PID -f /dev/null
```

The first line is the one-liner we analyzed in the previous sections.

Then, note how in the second line we save the pid of the remote iperf client into `IPERF_PID` by using `$!` after a command ending in `&`.

We need this pid so we can wait for iperf to finish using the `tail` command.

In the `[...]` section, you could include anything you want to do after you started running iperf. In the past, I had some accessor programs that started collecting data during iperf execution but those have been deprecated and are just left in for *\~inspiration\~*.
`
```shell
# Perform Latency Test 
sleep 3
if test -f /proc/latency_module; then
	python3 latency_accessor.py $current_path/data/$exp_name/latency $type
fi
if test -f /proc/ipi_lat_module; then
	python3 $current_path/module/ipi_latency_module/accessor.py $current_path/data/$exp_name/latency $type
fi
```

And that's the application execution portion of the script. I'll probably start lowering the level of detail in the next parts so that this document does not get too long, but I thought it was important that you fully understand how the application is run.

# Performing the System Configurations

# Orchestrating Data Collection

# Argument List

## Experiment Name - *exp_name*

## RSS support - *rss*

## RPS support - *rps*

## RFS support - *rfs*

## IAPS support - *iaps* (custom only)

## IAPS steering configuration - *backup_core* (custom only)

## Connection Number - *conns*

## Target Network Interface - *intf*

## IAPS busy list configuration - *iaps_busy_list* (custom only)

## RX Queue Number - *num_queue* 

## GRO Support - *gro*

## App and Network Isolation - *separate*

## Maximum Segment Size - *mss*

## Beginning of CPU Core Range - *core_start*

## Number of Total CPU Cores - *core_num*

## iperf Experiment Duration - *time*



<<|[[Experiment Wrapper|next]]>>