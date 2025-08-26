---
title: Experiment Wrapper
draft: false
tags:
---
DONE

The experiment wrapper is a script called `run_wrapper.sh` and was created to provide a more well-defined interface to the [[Single Experiment Script]]. The [[Single Experiment Script]] takes positional arguments, instead of options-based arguments, which got out of control as more and more arguments were added.

The Experiment Wrapper provides an option-based argument interface and translates the given arguments into positional arguments for the [[Single Experiment Script]]. This makes it a lot easier to run small-scale experiments directly from the command line without having to memorize the default values and positions of all possible variables.

The Experiment Wrapper starts with a list of default values for every experiment together with a comment on which option it corresponds to.

```bash
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
```

Then it gets the options using `getopts`

```bash
while getopts ":P:b:c:i:q:G:s:m:S:C:t:n:" opt; do
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
    \?) echo "Invalid option -$OPTARG" >&2
    exit 1
    ;;
  esac
```

Then, finally, it calls the [[Single Experiment Script]] with all the arguments. If they were given by the user through the options-interface, they will have the user-defined value. Otherwise, the default value is passed.

```bash
$current_path/run_mini_project.sh $EXP_NAME $RSS $RPS $RFS $IAPS $Backup_Core $Conns $INTF $IAPS_BUSY_LIST $NUM_QUEUE $GRO $SEPARATE $MSS $CORE_START $CORE_NUM $TIME
```

The experiment wrapper is very straight-forward and does not require much explanation. For detailed info on how to add new experiment parameters into the experiment wrapper, refer to [[Adding New Experimental Parameters]].