---
title: Adding New Experimental Parameters
draft: false
tags:
---
DONE
  
When adding a new experimental parameter it is important to keep in mind where that parameter is set vs. where the actual setup/application is run that is affected by the parameter.

My infrastructure goes through multiple steps of indirection before actually using a parameter. The order is something like this:

[[Setup Iterator|Setup Iterator]] -> [[Experiment Iterator]] -> [[Experiment Wrapper]] -> [[Single Experiment Script|Experiment Script]]

For now, lets exclude the act of actually using your parameter. First, let's just pass a parameter through the appropriate channels into the experiment script.

For this, you will have to take the following steps:

1. Add your parameter to the list of arguments in the experiment script and choose a sensible default
2. Add an option for your parameter into the experiment wrapper 
3. Write your setter-function into the library to make it accessible for new setup iterators

Let's show this process using an example.

When running experiments in a virtual environment, I always struggle to get traffic into my virtual machine, but getting traffic out of it is simple.
Usually, when running my experiments, the host node is the server and the remote node is the client. However, in a virtual environment, it is easier for me to set the host node as the client, the remote node as the server, and run iperf in reverse mode (having the server send to the client).

Currently, this option is commented out in my script and I would uncomment it every time I ran experiments in a virtual environment. Let's turn it into a parameter!

# Adding the parameter to the experiment script

The `run_mini_project.sh` script has gotten completely out of control with its arguments (hence the wrapper). 

``` shell
exp_name=${1:-exp}
rss=${2:-1}
rps=${3:-0}
rfs=${4:-0}
custom=${5:-0}
backup_core=${6:-1}
conns=${7:-6}
intf=${8:-ens4np0}
iaps_busy_list=${9:-0}
num_queue=${10:-8}
gro=${11:-1}
separate=${12:-0}
mss=${13:-1460}
core_start=${14:-0}
core_num=${15:-8}
time=${16:-10} 
```

When adding a new parameter, add it at the end with the default value. Like that, the compatibility of older experiment suites that might rely on the current order won't be compromised.

Like so:
```shell
exp_name=${1:-exp}
rss=${2:-1}
rps=${3:-0}
rfs=${4:-0}
custom=${5:-0}
backup_core=${6:-1}
conns=${7:-6}
intf=${8:-ens4np0}
iaps_busy_list=${9:-0}
num_queue=${10:-8}
gro=${11:-1}
separate=${12:-0}
mss=${13:-1460}
core_start=${14:-0}
core_num=${15:-8}
time=${16:-10} 
virtual=${17:-0}
```

>[!question]- What should be my default?
> When you are unsure about finding a default value, think about which value would be least disruptive to all the experiments that used the script prior to you adding your parameter. Here, we choose virtual to be 0, because all standard experiments so far where run in a non-virtual environment, so setting the default to virtual=1 would break all those experiments. 
> Sometimes, there might not be a good default, because you always need a custom value. In that case make the argument mandatory and exit the script with an appropriate error message when no argument is passed.

That's everything you have to do in the experiment script for now. (We'll have to come back later for the actual integration of the parameter.)

# Adding your parameter to the wrapper

The wrapper's purpose is to change the experiment script's position-based arguments into option-based arguments. Before the wrapper, when you wanted to change the last argument of the experiment, you'd have to put in all the prior arguments as well, which turned into a mess of 0s and 1s. The wrapper takes arguments as options and knows about all the defaults and the argument order of the experiment script. It was a huge game changer when writing new experiments.

Anyways, what do you need to add to the wrapper when adding a new parameter? Just a variable holding the default and the option parsing for your parameter.

The defaults are written at the top. Let's add our virtual parameter. Also lets give it the option `-v`.
``` shell
# [DEFAULTS]

EXP_NAME=exp 		# -n
RSS=1
RPS=0
RFS=0
IAPS=0
# -P (bitmask) 1   0   0   0
#			   ^   ^   ^   ^
#			   |   |   |   |
#			  RSS RPS RFS IAPS

Backup_Core=1		# -b
Conns=6 			# -c
INTF=ens4np0		# -i
IAPS_BUSY_LIST=0	# DEPRECATED
NUM_QUEUE=8			# -q
GRO=1				# -G
SEPARATE=0			# -s
MSS=1460			# -m
CORE_START=0		# -S
CORE_NUM=8			# -C
TIME=10				# -t
VIRTUAL=0           # -v
```

Now, we need to add the option parsing.
Add `v:` to the getopts string at the top and the actual implementation at the bottom with the others.
```shell
while getopts ":P:b:c:i:q:G:s:m:S:C:t:n:v:" opt; do
  case $opt in
    P) BITMASK="$OPTARG"
	   RSS=$(( (2#$BITMASK & 2#1000) >> 3))
	   RPS=$(( (2#$BITMASK & 2#0100) >> 2))
	   RFS=$(( (2#$BITMASK & 2#0010) >> 1))
	   IAPS=$((2#$BITMASK & 2#0001))
    ;;
    b) Backup_Core="$OPTARG"
    ;;
    c) Conns="$OPTARG"
    ;;
    i) INTF="$OPTARG"
    ;;
    q) NUM_QUEUE="$OPTARG"
    ;;
    G) GRO="$OPTARG"
    ;;
    s) SEPARATE="$OPTARG"
    ;;
    m) MSS="$OPTARG"
    ;;
    S) CORE_START="$OPTARG"
    ;;
    C) CORE_NUM="$OPTARG"
    ;;
    t) TIME="$OPTARG"
    ;;
    n) EXP_NAME="$OPTARG"
    ;;
    v) VIRTUAL="$OPTARG"
    ;;
    \?) echo "Invalid option -$OPTARG" >&2
    exit 1
    ;;
  esac

  case $OPTARG in
    -*) echo "Option $opt needs a valid argument"
    exit 1
    ;;
  esac
done
```

Now, lastly, we add our `VIRTUAL` value to the call of our actual experiment script at the bottom.

```shell
$current_path/run_mini_project.sh $EXP_NAME $RSS $RPS $RFS $IAPS $Backup_Core $Conns $INTF $IAPS_BUSY_LIST $NUM_QUEUE $GRO $SEPARATE $MSS $CORE_START $CORE_NUM $TIME $VIRTUAL
```

Your new parameter now has been added into the wrapper and will be passed to the experiment script!

# Exposing your parameter through my_lib.sh

Now, the last step is integrating your parameter into the library for easy access from the setup iterators.  

The library functions all work in a way that they get a value from the user and append that value to an ever-growing argument string with the correct option. That argument string will eventually be passed to the wrapper.

For our `virtual` parameter, we should write a function like this:
```shell
set_virtual(){
	virtual=$1
	ARG_STRING="$ARG_STRING -v $virtual"
}
```

Now it is ready to be used in whatever setup iterator you write!

# Integrating the parameter into the script

Now, there is absolutely no rules to this. It will entirely rely on what your parameter does. As long as you know basic bash scripting, you can throw it into the script.

For the sake of completing the example of adding virtual support, let's add the actual implementation.


```shell
if [[ "$virtual" == 0 ]]
then
	taskset -c "$APP_CORE-$((APP_CORE + APP_CORE_NUM - 1))" $IPERF_BIN -s -1 -J $IPERF_CUSTOM_ARGS > $current_path/iperf.json & ssh $remote_client_addr "iperf3 -c ${server_ip} -P ${conns} -M ${mss} -t ${time} > /dev/null"&
else
	# For use in virtual environment, we use the reverse iperf setting (no custom iperf is usable)
	ssh $remote_client_addr "iperf3 -s -1 > /dev/null" & taskset -c "$APP_CORE-$((APP_CORE + APP_CORE_NUM - 1))" $IPERF_BIN -c 10.0.0.4 -J -P ${conns} -M ${mss} -t ${time}> $current_path/iperf.json &
fi

```

Note how the remote client's 100G interface IP is hardcoded though... Its not pretty, but it works (for my setup and my setup only...)