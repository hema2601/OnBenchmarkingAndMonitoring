---
title: Experiment Iterator
draft: false
tags:
---
DONE

The experiment iterator just has the task of dispatching an experiment with the same setup for a user-defined number of times while also changing a main experiment variable.

This main experiment variable is always the number of connections, but changing the code so that that can be changed would be a good idea.

The experiment iterator is so small that it does not have its own file, but is implemented as part of the shared library file `my_lib.sh`.

```bash
run_exp () { #1: Name 2:Rep 3:Conn 4:Exp
	rep=$2
	conns=$3
	as_exponential=$4
    conn=1
    for((i=1;i<=$conns;i++));
    do
		if [[ "$as_exponential" == 1 ]]
		then
			marker=$conn
		else
			marker=$i
		fi
        for((j=1;j<=$rep;j++));
        do
            dir="$1"_"$marker"_"$j"
			$current_path/run_wrapper.sh $ARG_STRING -n $dir -c $marker
        done
        conn=$((conn*2))
    done

}
```

The `run_exp` function is called by the [[Setup Iterator]]. It requires the name of the experiment, how often each experiment should be run, the number of connection increases, and whether or not the connection should be increased exponentially.

Based on those parameters, the experiment iterator defines the directory name of the experiment, which consists of the experiment name, the current connection number, and the current repetition.

It passes that name together with the connection number and an `ARG_STRING` to the [[Single Experiment Script]]. The `ARG_STRING` is a global variable built up in the [[Setup Iterator]] using functions from `my_lib.sh` and includes information on the necessary environmental setup, which is then performed in the [[Single Experiment Script]].


