---
title: Single Experiment Script
draft: false
tags:
---


The single experiment script is the heart of the testing infrastructure.

This is because in the beginning, my 'testing infrastructure' only consisted of this very bloated piece of code. Eventually, I started to build the other components around it to make everything more manageable and configurable. 

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
iperf's output is then redirected into an iperf.json file in the `current_path` and we use the `&` to detach the iperf process from the command line, so we don't wait for it to finish.

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
> The binary of my custom iperf being called `iperf3_napi` is not something that works out-of-the-box when installing from my git repository. When installing custom iperf, I only compile it locally using `make` and created a symlink using `ln` in `/usr/bin` called `iperf3_napi` that points to the compiled binary in my local repository. Do not run `make install` on my custom iperf because it might overwrite the normal iperf and I have not tested my iperf extensively enough to recommend this.

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

As explained in the [[Adding New Data Sources|data source tutorial]], there are essentially three types of data collection in my experiment script:
1. Counters
2. Background Programs
3. Application

The application data collection was already explained in the [[#Running the Target Application]] part, since it is simply the output from the iperf execution.
I will explain the data collection of the others in the following parts.
## Counters

All the counter data used in the experiment is captured with the help of the `before.sh` and `after.sh` scripts. As the name suggests, `before.sh` writes the state of the counters before the experiment into a temporary file and `after.sh` does the same with the counters after the experiment. Then, the `after.sh` script also concats the two temporary files into a raw data file that is then going to be converted into json data using the [[Raw Data Converter]].

As of writing this document, there are 10 raw data files compiled from counters, 4 of which are taken from a custom proc file created by the IAPS module. I will go over the other 6, where their data is taken from and what the data describes.

### ethtool counters

ethtool is a tool with an abundance of functionalities, but in essence, it is there to communicate with network interfaces. When using the `-S` option, it prints out various statistics. For my experiments, I needed the number of packets that arrived at my NIC and the number of packets that were dropped. This was achieved with the code below.

```bash
ethtool -S ${intf} | grep 'packets\|dropped:' > before_pkt.txt
```

>[!warning]- Portability of ethtool statistics
>The naming of the ethtool counters is not standardized, but rather up to the company providing the drivers. The below code for example only works for Intel NICs, because the names of the same stats published through a Mellanox driver are different. If you run experiments in a new environment and you are having issues with your ethtool-collected data (either in the [[Raw Data Converter]] or during visualization), check your ethtool output!
### softirq counters

In the proc subsystem, there is a file called `softirqs` that keeps a count on all types of softirqs and on which core they were triggered. The contents look like this:
![[Pasted image 20250828081514.png]]
For my experiments, I was only interested in the number of softirqs involving networking, so I filtered them out using `grep`. 

```bash
cat /proc/softirqs | grep NET_ > before_soft_irq.txt 
```

### irq counters

This counter is the only one that is not a one liner. Normal hardware interrupts are counted in a similar fashion to softirqs. However, instead of having 10 types of interrupts like the sotfirqs, there can be many more hardware interrupts in your system, depending on your hardware specs and connected peripheral devices. Therefore, I first write the list of interrupts associated with my network interface into a temporary file and then only save the counters associated with those interrupts into the data file. It would also be possible to just write all the interrupts down and do the filtering within the [[Raw Data Converter]].

```bash
$current_path/scripts/print_irq_cnt.sh $intf > tmp.txt
cat /proc/interrupts | grep -f tmp.txt > before_irq.txt
```

The below is an example of the output of the `/proc/interrupts` file on a relatively small system. The numbers at the front are the numbers that every interrupt is identified with. On the right side are some high-level information, such as associated device and pcie addresses. 
![[Pasted image 20250828083615.png]]

### softnet counters

The counters in `/proc/net/softnet_stat` provide information on packets that leave the netdevice subsystem into the actual network stack. On a default system, this is a rather boring counter. As you can see below, it is mainly just a bunch of zeros.
![[Pasted image 20250828153731.png]]
The data in softnet_stat becomes interesting once you start experimenting with software-based packet steering like RPS or RFS.
Let's take a look what the different counters represent. Keep in mind that one row corresponds to the set of counters for one CPU core.

>[!warning] The values are in hexadecimal! Keep that in mind when parsing!
>
>

1st Column: Processed Packets
	This is the only counter that is incremented on a normal system. It gets incremented in `__netif_receive_skb_core` (specifically [here](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L5782)). This represents the number of GRO-aggregated packets entering the stack. Therefore, it will be different from the number of packets you would see reported by something like `ethtool`.

2nd Column: Dropped Packets
	This counter represents the number of packets dropped at **the softnet backlog**. Do not confuse it with packet drops at the NIC (Use `ethtool` for that). If this column is anything other than 0, you should reconsider your setup. The default maximum length of the backlog (which is used during software-based packet steering) is [configured to be 1000](https://elixir.bootlin.com/linux/v6.16/source/net/core/hotdata.c#L17). So if this counter is not 0, either you configured the backlog length to be very low, or your software queues are building up beyond 1000 packets, which would be very bad. Another third option is that a packet was dropped due to exceeding the CPU's flow limit, which is further explained in the 11th Column part.

3rd Column: Time Squeeze
	A 'time squeeze' in this situation refers to the scenario when one polling cycle of NAPI exceeds its allocated runtime. This happens in [two scenarios](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L7611): Either when it has exhausted its [packet limit of 300 packets](https://elixir.bootlin.com/linux/v6.16/source/net/core/hotdata.c#L12), or it exceeded its [time limit of 2 jiffies](https://elixir.bootlin.com/linux/v6.16/source/net/core/hotdata.c#L14). In my experience, this counter increases very rarely. The napi structs used during software-based packet steering are [initialized to the value of `weight_p`](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L12826), which is [set to 64](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L4787), so their limit is well below the 300 packet limit. The napi structs used for the initial packet processing are defined by the drivers, so they might exceed the limit, but it is safe to assume that they will operate within a sensible limit. Most likely when the time squeeze counter is increased, it will be because some packet took abnormally long to be processed and therefore NAPI ran out of time.

4th to 9th Column: Zero
	These columns are hard-coded to be 0. I used some of these columns to publish my custom data counters without having to set up a new proc file.

10th Column: Received RPS
	This column counts how often a core received an RPS request. In other words, this is how often a core was notified through an IPI to start processing packets from the backlog. It is increased [here](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L5035).

11th Column: Flow Limit Count
	I haven't worked with this counter a lot. To the best of my understanding, when using software-based packet steering, specifically when using RFS, every core is assigned a limit of how many concurrent flows it is allowed to handle. If a new skb arrives and causes the number of concurrent flows handled by the CPU core to overflow, the [counter is increased](https://elixir.bootlin.com/linux/v6.16/source/net/core/dev.c#L5132). In this case, the packet is dropped and the counter in column 2 will be increased as well. I don't think I have ever seen this counter increase in my experiments, so I never bothered to fully hunt down the logic of flow limits in the code, so take this explanation with a grain of salt.

12th Column: Combined Queue length of the backlog
	A full explanation of the queue logistics of the backlog would be a bit much at this point. Just know this: Packets on the backlog can either be on the `process_queue` - the place where packets are *actively processed* - or on the `input_pkt_queue` - the place where packets await active processing. The combined length of those queues is stored in the 12th column.

13th Column: CPU number
	*Finally*. After 12 index-less hexadecimal numbers somebody thought of adding an index to this proc file and probably wasn't able to add it at the beginning, because it would break things. 

14th Column: input queue length
	This value represents the number of packets currently awaiting active processing on the backlog. This value together with the value from the 15th column add up to the 12th column.

15th Column: process queue length
	This value represents the number of packets being actively processed by this CPU from the backlog. This value together with the value from the 14th column add up to the 12th column.

If you want to check for yourself, the proc file is printed [here](https://elixir.bootlin.com/linux/v6.16/source/net/core/net-procfs.c#L145).

In the `before.sh` script, the values are captured as shown below.
```bash
cat /proc/net/softnet_stat > before_softnet.txt
```
### proc/stat counters

The `/proc/stat` includes very fundamental counters. To the best of my knowledge, tools like `top` use it to display the CPU usage percentages. I never really looked in detail into how and where its individual counters are increased in the code, I just used resources online to understand them. Luckily, since it is such a fundamental file, there is decent documentation on it, like this [man page](https://man7.org/linux/man-pages/man5/proc_stat.5.html).
I used the values from `/proc/stat` to get an idea for what work my individual CPUs were performing, to see whether my packet steering was working properly. 

The data was collected as below:
```bash
cat /proc/stat > before_proc_stat.txt
```

>[!warning]- Do not treat my /proc/stat usage as a reliable example
>When you look at the way I visualized this data (`web/components/cpu_util_graph.js`), you will see that I trial-and-errored it a little bit. My CPU utilization was never adding up to 100%, so I started subtracting 200 from the idle cycles. I do not remember whether I had better reasoning than "somehow the values look good when I decrease the idle cycles by 200". I always referred to my CPU utilization visualization with caution for that reason. If you want to get proper usage examples, refer to the implementation of tools like `top`, `htop`, or `sar`, which all use this proc file, as far as I know.

### netstat counters

Finally, we have the netstat counters from `/proc/net/netstat`. This counter file is nice for parsing but terrible for reading. It looks like this:
![[Pasted image 20250828173704.png]]

What you can see are counters corresponding to well-defined networking events. The lines starting with `TcpExt:` show TCP-related events and the lines starting with `IpExt:` show IP-related events. Based on your system, you might see more contents. For each pair of lines, the following relation applies: The first line gives a list of event names, the second line gives the counters corresponding to the events from the first line in the same order.

As you can see, I simply read the entire contents into a raw data file, because parsing it in a bash script would be too complicated. I then reduce the number of counters when the file is processed by the [[Raw Data Converter]].

```bash
cat /proc/net/netstat > before_netstat.txt
```

For my experiments, I only used three coutners:
1. TCPOFOQueue - counts the number of packets that were enqueued on the out-of-order queue
2. TCPHPHits - Counts the number of times a packet was able to take the 'fast path' in TCP
3. TCPOFODrop - Counts the number of times a packet was dropped because the out-of-order queue did not have any more space

If you are interested in any of the other events, they are documented rather well in the kernel itself under [/Documentation/networking/snmp_counter.rst](https://elixir.bootlin.com/linux/v6.16/source/Documentation/networking/snmp_counter.rst)

## Background Programs

To say 'background program**s**' is a bit of an overstatement. At this point, the experiment script only runs one background program: `perf stat`.

## perf stat

`perf stat` collects counters on specified events (can be hardware counters or user-defined, so very powerful). I use it in my testing to count the number of instructions, cycles, LLC accesses, and LLC misses on the cores that my experiment runs on.

```BASH
$PERF_BIN stat -C $core_start-$((core_start + core_num - 1)) -e cycles,instructions,LLC-loads,LLC-load-misses -o $current_path/perf_stat.json &
PERFSTAT_PID=$!
# ==[run experiment]==
[...]
# ====================
kill -s SIGINT $PERFSTAT_PID
tail --pid=$PERFSTAT_PID -f /dev/null
```

In the first line, we run our `perf stat` by telling it which core it should count on (-C), what events to look out for (-e), and what file to write to (-o)
Then we end the line with an `&`. What this does is that it detaches the process from the command line. Usually, when you run anything, the console will wait for it to finish, but if you put the `&`, it just detaches and throws the pid at you.
![[Pasted image 20250806173359.png|500]]
In the second line, we catch that pid using `$!` and save it in a variable so that we can kill the process later when the experiment is done.

After the experiment has finished, we send the `SIGINT` signal to our saved pid. This is the same signal that is sent when pressing Ctrl-C on your keyboard.

Lastly, with the `tail` instruction, we simply keep our shell script from running away before our process has successfully been killed. If you ever interrupted a heavy profiling program like `perf stat`, you know that sometimes it takes a little while to write all of its counters into the right output.

## perf stat on separate cores

In the case that we run the separate tasks of the experiment on [[#App and Network Isolation - *separate*|separate]] cores, `perf stat` becomes a little more complicated:
```bash
if [[ "$separate" == "1" ]]
then
	$PERF_BIN stat -C $IRQ_CORE-$((IRQ_CORE + IRQ_CORE_NUM - 1)) -e cycles,instructions,LLC-loads,LLC-load-misses -o $current_path/perf_stat_irq.json &
	IRQPERFSTAT_PID=$!
	if [[ "$PP_CORE_NUM" != "0" ]]
	then
		$PERF_BIN stat -C $PP_CORE-$((PP_CORE + PP_CORE_NUM - 1)) -e cycles,instructions,LLC-loads,LLC-load-misses -o $current_path/perf_stat_pp.json &
		PPPERFSTAT_PID=$!
	fi
	$PERF_BIN stat -C $APP_CORE-$((APP_CORE + APP_CORE_NUM - 1)) -e cycles,instructions,LLC-loads,LLC-load-misses -o $current_path/perf_stat_app.json &
	APPPERFSTAT_PID=$!
fi
```

We run multiple instances of `perf stat` for different CPU core groups and write them all to separate files. Then, when all data has been written, we accumulate those separate files into one big `perf stat` file:

```bash
if [[ "$separate" == 1 ]]
then
	if test -f $current_path/perf_stat_irq.json
	then 
		echo "TYPE	IRQ" >> $current_path/data/$exp_name/perf_stat.json 
		cat $current_path/perf_stat_irq.json >> $current_path/data/$exp_name/perf_stat.json
		rm $current_path/perf_stat_irq.json
	fi

	if test -f $current_path/perf_stat_pp.json
	then 
		echo "TYPE	PP" >> $current_path/data/$exp_name/perf_stat.json 
		cat $current_path/perf_stat_pp.json >> $current_path/data/$exp_name/perf_stat.json
		rm $current_path/perf_stat_pp.json
	fi

	if test -f $current_path/perf_stat_app.json
	then 
		echo "TYPE	APP" >> $current_path/data/$exp_name/perf_stat.json 
		cat $current_path/perf_stat_app.json >> $current_path/data/$exp_name/perf_stat.json
		rm $current_path/perf_stat_app.json
	fi

fi

echo "TYPE	FULL" >> $current_path/data/$exp_name/perf_stat.json 
cat $current_path/perf_stat.json >> $current_path/data/$exp_name/perf_stat.json
rm $current_path/perf_stat.json
```

The result will be one big `perf_stat.json` file with the following structure:

```txt
TYPE IRQ

...[contents of the perf stat data on irq cores]...

TYPE PP

...[contents of the perf stat data on packet processing cores]...

TYPE APP

...[contents of the perf stat data on application cores]...

TYPE FULL

...[contents of the perf stat data on all cores]...
```

As you might know by now, this file will then be given to the [[Raw Data Converter]] and it will format it into a nicely parse-able json file.

## Raw Data Converter Dispatch

There is not much to say about this part of the script. The [[Raw Data Converter]] is triggered for every single experiment within the experiment script. As arguments it gets the list of raw data files you want to convert. 

```bash
python3 $current_path/file_formatter.py $exp_name IRQ SOFTIRQ PACKET_CNT IPERF SOFTNET PROC_STAT PKT_STEER PERF_STAT IPERF_LAT BUSY_HISTO PKT_LAT_HISTO NETSTAT PKT_SIZE_HISTO
```

# Argument List

## Experiment Name - *exp_name*

**Default**: "exp"

The experiment name is the name of the sub-experiment folder where all the raw data files will be saved to. The experiment script allocates it in the `data/` folder.

```bash
#create directory
mkdir $current_path/data/$exp_name
```

This variable is further used to:
1. Tell the `after.sh` scripts where to save the concatted data files to
2. Move the iperf output into the sub-experiment folder
3. Compile the separate `perf stat` outputs into one file in the sub-experiment folder
4. Give a target to the [[Raw Data Converter]]
## RSS support - *rss*

**Default**: 1

This argument indicates whether to use RSS or not. Turning off RSS is done by reducing the number of RX queues of our network interface to 1. When RSS is turned on, the number of RX queue is set to [[#RX Queue Number - *num_queue*|num_queue]]

```bash
if [[ "$rss" == "1" ]]
then
	ethtool -L $intf combined $num_queue
	type="RSS"
else
	ethtool -L $intf combined 1
	num_queue=1
fi
```

## RPS support - *rps*

**Default**: 0

This argument decides whether RPS is enabled or not. When it is set to 1, a script enabling RPS is run, otherwise a script disabling RPS is run. To enable RPS, a bitmask of cores that can be RPS steering target needs to be assigned. For that, the information in `PP_CORE` and `PP_CORE_NUM` is used. Those two describe the range of packet processing cores and are set based on [[#App and Network Isolation - *separate*|the 'separate' argument]].

```bash
if [[ "$rps" == "1" ]]
then
	#enable rps
	$current_path/scripts/enable_rps.sh $intf $PP_CORE $PP_CORE_NUM
	type="RPS"
else
	$current_path/scripts/disable_rps.sh $intf
fi
```

RPS and RSS can be active at the same time.
RPS and RFS should not be activated at the same time, since I do not exactly know what happens in that situation.

For more info on how to actually activate/deactivate RPS, refer to the `enable_rps.sh` and `disable_rps.sh` scripts in the `scripts/` folder.
## RFS support - *rfs*

**Default**: 0

This argument enables or disables RFS. It runs the `enable_rfs.sh` script when RFS is supposed to be turned on and `disable_rfs.sh` when it is supposed to be turned off. 

```bash
if [[ "$rfs" == "1" ]]
then
	#enable rfs
	$current_path/scripts/enable_rfs.sh $intf
	type="RFS"
else
	$current_path/scripts/disable_rfs.sh $intf
fi
```

RFS does not work properly together with RSS.
RFS and RPS should not be activated at the same time, since I do not exactly know what happens in that situation.

For more info on how to actually activate/deactivate RFS, refer to the `enable_rfs.sh` and `disable_rfs.sh` scripts in the `scripts/` folder.
## IAPS support - *custom* (custom only)

**Default**: 0

>[!warning]- Incompatible with unmodified system
>If you want to make use of IAPS, you would have to install a custom kernel and load the IAPS module. If you try activating this feature on an unmodified system, this script will most likely throw an error.

This argument enables or disables IAPS. In case of disabling IAPS, the script first checks if the system even supports IAPS. If yes, it is turned off, otherwise the argument is ignored.
IAPS and its parameters are configured through module arguments. Information on the [[#IAPS steering configuration - *backup_core* (custom only)|backup core]] and [[#IAPS busy list configuration - *iaps_busy_list* (custom only)|busy list]] arguments will be in other parts. The `base_cpu` and `max_cpus` parameters need to be set to determine which CPUs can be chosen as steering targets by IAPS. Note that IAPS only supports range of CPUs, not detached CPU sets. Lastly, the `custom_toggle`  parameter is set to 1 to actually activate IAPS.

Also, IAPS relies on other software-based packet steering schemes to be enabled. Therefore, we enable RPS by default when using IAPS and if IAPS requires RFS support, RFS will be enabled as well. Here, it is okay to enable RPS and RFS at the same time, because IAPS overwrites their actual steering algorithms. We only need RFS for the data structures it provides.

I never checked whether IAPS relying on RFS works in a multiqueue setup together with RSS. 

```bash
if [[ "$custom" == "1" ]]
then

	$current_path/scripts/enable_rps.sh $intf $PP_CORE $PP_CORE_NUM	
    echo $backup_core > /sys/module/pkt_steer_module/parameters/choose_backup_core
	
	if [[ "$backup_core" == "1" ]]
	then
		$current_path/scripts/enable_rfs.sh $intf
	fi
	echo $iaps_busy_list > /sys/module/pkt_steer_module/parameters/list_position
	echo $PP_CORE > /sys/module/pkt_steer_module/parameters/base_cpu
	echo $PP_CORE_NUM > /sys/module/pkt_steer_module/parameters/max_cpus
	echo 1 > /sys/module/pkt_steer_module/parameters/custom_toggle
	type="IAPS"
else
	echo "Disable Custom"
	if test -f /sys/module/pkt_steer_module/parameters/custom_toggle
	then 
		echo 0 > /sys/module/pkt_steer_module/parameters/custom_toggle
	fi
fi
```
## IAPS steering configuration - *backup_core* (custom only)

**Default**:1

>[!warning]- Incompatible with unmodified system
>If you want to make use of IAPS, you would have to install a custom kernel and load the IAPS module. If you try activating this feature on an unmodified system, this script might throw an error.

IAPS needs to choose a backup core in the case that its preferred steering target is unavailable. There are five different backup core choices.

1: Application Core
Rely on RFS data to steer the packet to the last application core

2: Current Core
Just send the packet to the current core to avoid triggering an interrupt altogether

3: Hash-based
Decide the next target by using the RPS hash over the available CPU set.

4: Load Balancing
Custom Load Balancing approach of IAPS. In its current state it would run the experimental Idle Core Activation 

5: Previous Core
Steer the packet to the previous target core

Though the default value is 1, the most performant and most used option for this would be 3.

It is set using a module parameter during the IAPS activation phase.

```bash
echo $backup_core > /sys/module/pkt_steer_module/parameters/choose_backup_core
```
## Connection Number - *conns*

**Default**: 6

Number of connections spawned by the iperf client.

```bash
[...] ssh $remote_client_addr "iperf3 -c ${server_ip} -P ${conns} -M ${mss} -t ${time} > /dev/null"&
```

## Target Network Interface - *intf*

**Default**: ens4np0

This is the name of the network interface on the system that you are running the experiment on. Note that this will be the name of your high-throughput interface where you are going to receive the actual iperf traffic through, not your motherboard's 1G interface. 

The interface name is important to configure interface-specific parameters such as [[#RX Queue Number - *num_queue*|RX queues]] or [[#GRO Support - *gro*|GRO]], or to collect data through `ethtool`.

## IAPS busy list configuration - *iaps_busy_list* (custom only)

**Default**: 0

>[!warning]- Incompatible with unmodified system
>If you want to make use of IAPS, you would have to install a custom kernel and load the IAPS module. If you try activating this feature on an unmodified system, this script might throw an error.

You can consider this argument as pretty much depricated, but for completion's sake: IAPS maintains busy cores on a busy list. At some point, I was experimenting with whether it would make a difference to choose busy cores from the tail of the list, instead of the head. In short: It didn't.

This argument is left in to not break any of my old setup iterator scripts, but the module parameter it is setting has been removed from IAPS's code.
```bash
echo $iaps_busy_list > /sys/module/pkt_steer_module/parameters/list_position
```

## RX Queue Number - *num_queue* 

**Default**: 8

This argument decides the number of RX queues that are allocated on the network interface. Note that no matter the value of `num_queue`, in the case that RSS is deactivated, it will be set to 1. The number of RX queues is set using `ethtool`. 

The number of RX queues also affects the distribution of CPU cores in the [[#App and Network Isolation - *separate*|separated]] task scenario, which will be further explained there.
## GRO Support - *gro*

**Default**: 1

This argument decides whether GRO will be turned off or not. It is turned on by default. GRO support is enabled or disabled using `ethtool` for the specified network interface.

```bash
if [[ "$gro" == "1" ]]
then
	ethtool -K $intf gro on
else
	ethtool -K $intf gro off
fi
```
## App and Network Isolation - *separate*

**Default**: 0

This argument indicates whether separate processing tasks are to be isolated from each other or not. There are three separate processing tasks in this experiment:

APP: The iperf application 
IRQ: The interrupt processing of incoming packets
PP: The packet processing of packets that have been steered to a new core in software

Only experiments using RPS or IAPS have all three.
When using RSS, there is no PP task, because the packet processing is performed directly on the IRQ CPU. When using RFS, there is no possible PP and APP isolation, because RFS steers packets to be processed on the same core as the application by definition.


## Maximum Segment Size - *mss*

**Default**: 1460

## Beginning of CPU Core Range - *core_start*

**Default**: 0

## Number of Total CPU Cores - *core_num*

**Default**: 8

## iperf Experiment Duration - *time*

**Default**: 10


