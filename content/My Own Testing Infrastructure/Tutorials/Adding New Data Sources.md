---
title: Adding New Data Sources
draft: false
tags:
---
 
When adding a new data source there are two steps to be considered:

1. Integrate the automatic data collection into the experiment script
2. Take the data from your generated data file and integrate it into the post processing

Step 1 is the more difficult and less general step, so lets start with step 2.

## 1.1 Adding the processing of a new raw data file

Lets assume the experiment execution generated a new file called `new_data.json`.
**This file needs to be called `.json` even if it is - strictly speaking - only a `.txt` file**

### 1.1.1 Integrate into the Raw Data Converter

We first need to implement the translation of the raw data file to a well-formatted json file.
This is done in the [[Raw Data Converter]] (`file_formatter.py`).
#### 1.1.1.1 Add Enum and Json Generator Class into the Raw Data Converter
Any raw data file type, whose translation is defined in `file_formatter.py` needs to be added to the Filetype enum.
![[Pasted image 20250806152708.png]]

If you wanted to add the translation of `new_data.json`, you would have to do these **two** changes:

```python
class Filetype(Enum):
    [...]
    NETSTAT         = 13
    PKT_SIZE_HISTO  = 14
    NEW_DATA        = 15
Filetype = Enum('Filetype', [ [...], 'NETSTAT', 'PKT_SIZE_HISTO', 'NEW_DATA'])
```

Now that it has been added to the enum, you would need to actually implement the translation. For this, you need to define your own class that inherits `JsonGenerator`
It is not necessary to know all the details, just know that you need to add a class like this:

```python
class NEW_DATAGen(JsonGenerator):
	def generate_json(self):
        #print("Generate new_data.json")
        self.f.seek(0)
        self.f.truncate()
        json.dump(self.json_dict, self.f, indent=0)

    def read_source(self):
	    #implement translation here
```

Note that the naming is **not optional**. Your raw data file needs to be in lowercase, and the enum and the class need to be the exact name in uppercase, otherwise you will get an error.
The `generate_json` function can most likely be left alone. 
In the `read_source` function, you need to read your raw file (stored in `self.f`) and represent the data the way you want it in the `self.json_dict` empty list.
If you need inspiration, you can take a look at the existing Generators I have implemented.
#### 1.1.1.2 Example of a JsonGenerator implementation

Lets take a look at an example from my translations: `SOFTNETGen` 
(If you are unfamiliar with the json format or the json package in python, you'll need to understand those first)

```python
class SOFTNETGen(JsonGenerator):

    def generate_json(self):
        #print("Generate softnet.json")
        self.f.seek(0)
        self.f.truncate()
        json.dump(self.json_dict, self.f, indent=0)
        
    def read_source(self):
        #print("Read original softnet.json")
        for line in self.f:
            parts = [x for x in line.split(' ') if x.strip()]
            
            # Read Queue Number
            curr_cpu = int(parts[-3],16)
            
            # Check if object exists
            elem = next((item for item in self.json_dict if item["CPU"] == curr_cpu), None)
            
            # Write to object
            if elem is not None:
                elem["Processed"] = int(parts[0],16) - elem["Processed"]
                elem["Dropped"] = int(parts[1],16) - elem["Dropped"]
                elem["Squeezed"] = int(parts[2],16) - elem["Squeezed"]
                elem["RPS Interrupts"] = int(parts[9],16) - elem["RPS Interrupts"]
                elem["IPI Enqueued"] = int(parts[4],16)- elem["IPI Enqueued"]
                elem["InputQ Dequeued"] = int(parts[5],16)- elem["InputQ Dequeued"]
                elem["Input Pkts"] = int(parts[6],16) - elem["Input Pkts"]
                
            else:
                elem = dict()
                elem["CPU"] = curr_cpu
                elem["Processed"] = int(parts[0],16)
                elem["Dropped"] = int(parts[1],16)
                elem["Squeezed"] = int(parts[2],16)
                elem["RPS Interrupts"] = int(parts[9],16)
                elem["IPI Enqueued"] = int(parts[4],16)
                elem["InputQ Dequeued"] = int(parts[5],16)
                elem["Input Pkts"] = int(parts[6],16)
                self.json_dict.append(elem)
```

`self.f` is the `softnet.json` file opened using python. In the `read_source` function, the program goes over it line by line and translates each line's information according to my definition.

The file just contains the counters of `/proc/net/softnet_stats` before and after I ran iperf.

``` Shell
echo /proc/net/softnet_stats > before.txt
# ==[run experiment]==
[...]
# ====================
echo /proc/net/softnet_stats > after.txt
cat before.txt after.txt > softnet.json
```

With that in mind, we start iterating over each line of the file and break it up into easily accessible parts
```python
for line in self.f:
            parts = [x for x in line.split(' ') if x.strip()]
```

Then, we first check which CPU this line corresponds to:
The CPU number is the 3rd counter from the back and the value is written in hexadecimal.
```python
            curr_cpu = int(parts[-3],16)
```

Now because the `softnet.json` file contains both the before and after value for this CPU, we need to check if a line with this value has been parsed already:
(This one-liner was taken from stack overflow at some point. It just checks whether there is an item stored in the `json_dict` that has the same CPU value as our current line)
```python
            # Check if object exists
            elem = next((item for item in self.json_dict if item["CPU"] == curr_cpu), None)
```

If this element exists, we know that the current line represents the *after* value of our counter so we take the value that is already stored in our element and store the difference with our current value back into the object. This looks like this:
```python
# Write to object
            if elem is not None:
                elem["Processed"] = int(parts[0],16) - elem["Processed"]
                elem["Dropped"] = int(parts[1],16) - elem["Dropped"]
                elem["Squeezed"] = int(parts[2],16) - elem["Squeezed"]
                elem["RPS Interrupts"] = int(parts[9],16) - elem["RPS Interrupts"]
                elem["IPI Enqueued"] = int(parts[4],16)- elem["IPI Enqueued"]
                elem["InputQ Dequeued"] = int(parts[5],16)- elem["InputQ Dequeued"]
                elem["Input Pkts"] = int(parts[6],16) - elem["Input Pkts"]
```

If no element with this CPU number exists, we know that it is the first time we find a value for this CPU and therefore it is the *before* value.
For the first element, we simply create a new dictionary (necessary for json) and store our values into the dictionary with the names that we want. Then we append it into our `json_dict`:
```python
else:
                elem = dict()
                elem["CPU"] = curr_cpu
                elem["Processed"] = int(parts[0],16)
                elem["Dropped"] = int(parts[1],16)
                elem["Squeezed"] = int(parts[2],16)
                elem["RPS Interrupts"] = int(parts[9],16)
                elem["IPI Enqueued"] = int(parts[4],16)
                elem["InputQ Dequeued"] = int(parts[5],16)
                elem["Input Pkts"] = int(parts[6],16)
                self.json_dict.append(elem)
```

Now, after we have iterated over each line of the raw data and written it into our `json_dict`, we only have to write the contents from the `json_dict` into the file.

This happens in `generate_json`. This functions is the same for all my translations, but can be changed in case you want to add special formatting to your json output etc.
```python
def generate_json(self):
        #print("Generate softnet.json")
        self.f.seek(0)
        self.f.truncate()
        json.dump(self.json_dict, self.f, indent=0)
```
We jump to the front of our raw `softnet.json` file, then truncate it (deleting its contents), before we dump the contents of our `json_dict` into `softnet.json`.
This is how we change from a file that looks like this:
(truncated for brevity)
![[Pasted image 20250806160754.png|800]]
To this:
(also truncated)
![[Pasted image 20250806160949.png|500]]


#### 1.1.1.3 Make the experiment call you generator

Lastly, you need to tell `file_formatter.py` that you want your new generator to be run. For that, you need to go to the place where the program is called (in our case `run_mini_project.sh`)
```BASH
python3 $current_path/file_formatter.py $exp_name IRQ SOFTIRQ PACKET_CNT IPERF SOFTNET PROC_STAT PKT_STEER PERF_STAT IPERF_LAT BUSY_HISTO PKT_LAT_HISTO NETSTAT PKT_SIZE_HISTO
```
Just add your own option to the end of the list
```BASH
python3 $current_path/file_formatter.py $exp_name IRQ SOFTIRQ PACKET_CNT IPERF SOFTNET PROC_STAT PKT_STEER PERF_STAT IPERF_LAT BUSY_HISTO PKT_LAT_HISTO NETSTAT PKT_SIZE_HISTO NEW_DATA
```

Congratulations! You have implemented the translation from the raw data file to a well-formatted json.
### 1.1.2 Integrate into Summarizer

The integration into the [[Summarizer|summarizer]] is pretty straight-forward.
The summarizer doesn't need to know anything about your data, it only needs to know that it exists so that it can be summarized.
All you need to do is tell it that your file exists by adding its name to the file list.
Before:
```python
files = ["iperf.json", "irq.json", "packet_cnt.json", "softirq.json", "softnet.json", "pkt_steer.json", "latency.json", "proc_stat.json", "perf.json", "perf_stat.json", "iperf_lat.json", "busy_histo.json", "pkt_lat_histo.json", "netstat.json", "pkt_size_histo.json"]
```
After:
```python
files = ["iperf.json", "irq.json", "packet_cnt.json", "softirq.json", "softnet.json", "pkt_steer.json", "latency.json", "proc_stat.json", "perf.json", "perf_stat.json", "iperf_lat.json", "busy_histo.json", "pkt_lat_histo.json", "netstat.json", "pkt_size_histo.json", "new_data.json"]
```

Now, the summarizer will look for your file and summarize it if it finds it.
## 1.2 Adding the generation of a new raw data file to the experiment

Where to generate your new data file is going to depend on where your data is going to come from.
Lets go over the three main data sources used in my current infrastructure: Counters, background programs, and application data.

### 1.2.1 Counters

This is the simplest way of adding more data to your experiment and I strongly recommend you to try your best to get most of your data from counters.

Why?
1. Their implementation and integration is simple.
2. They have near-zero impact on your experiment's execution and are unlikely to affect your performance

The only downside with counters is that they are very low-resolution. When you run a 10 second iperf experiment, counters can tell you how many packets you received over the entire duration, but they cannot tell you whether there were any fluctuations between the packets received at second 1 vs second 8. 

The kernel exposes a lot of counters in its `/proc` subdirectory, so go exploring there, just don't hope to find any decent documentation...
When looking for existing counters in the system, it is often easiest to look for a tool that provides similar data and see where they get their data from. 9 out of 10 times, its some kernel counter. The other 1 out of 10 times, its perf.

For example, when I wanted to monitor how many cycles each CPU spends on different tasks (app vs system vs irq...), I checked how `top` gathers that information and found the `/proc/stat` file through that.

In the case that your wanted data is not provided as a counter, consider writing a kernel module that creates its own proc file and publish your own counters. Explaining how to write such a module will probably be contained in a separate entry, just keep in mind that attaching a module through something like fentry/fexit will have some overhead, and using something like kprobes is going to be be only useful for strictly debugging purposes due to the potential performance impact. Keep that in mind. Otherwise, just go crazy and put your counters into the kernel. If you check, you might find some deprecated entries in some proc files that you can safely write your data into. For example, `/proc/net/softnet_stats` has some fields that are hardcoded as 0. For my IAPS monitoring, I added some relevant counters directly into the kernel code and published them through the `softnet_stats` proc file, which worked great.

Anyways, you don't want a lecture on counters, you want to know how to integrate them.

Since they are so frequently used in my testing infrastructure, there are dedicated counter-recording scripts.
Those are `scripts/before.sh` and `scripts/after.sh`

`before.sh` records the counters before the experiment execution.
`after.sh` records the counters after the experiment execution, concatenates the before and after values into one file, moves the file into the proper directory, and finally deletes the before and after files.

Let's get a better idea of this by looking at how the data from `/proc/softirqs` is collected.

In `before.sh`, we record the counters like this:
```BASH
cat /proc/softirqs | grep NET_ > before_soft_irq.txt 
```
We pipe the contents of the file into grep to only select a few lines (in this case the network-related softirqs) and write it to our before file.

In `after.sh`, we perform the following steps:
```BASH
cat /proc/softirqs | grep NET_ > after_soft_irq.txt 
[...]
cat before_soft_irq.txt after_soft_irq.txt > $current_path/data/$name/softirq.json
[...]
rm before_soft_irq.txt
rm after_soft_irq.txt
```
We again save the counters, then we concatenate the two files and write it into `softirq.json` which is in our experiment directory, then we clean up.

If you have any counter - be that a counter file, or running something like `ethtool -S` - you simply add them to `before.sh` and `after.sh` and your file will be generated. Then you just implement the translation as explained [[#1. I want to add a data source#1.1 Adding the processing of a new raw data file|above]]

### 1.2.2 Background Programs

Sometimes, the wanted data comes from a program that needs to be started before the experiment runs and whose data needs to be collected after the experiment is done. 

These types of data sources need to be directly implemented into the experiment script.

Some examples of this would be `perf` or `sar`.

There is not as much to say about these background programs.
The only thing I want to add is that if your counters are not high-resolution enough for you, i.e. you want to collect more detailed counters at a more fine-grained time scale, you can implement a background program that periodically accesses your counters. Just know that the more background programs you run, the more noise there is on your system that might compromise your experiment.

To see how to integrate background programs into the experiment, lets look at the example of `perf stat`. `perf stat` collects counters on specified events (can be hardware counters or user-defined, so very powerful). I use it in my testing to count the number of instructions, cycles, LLC accesses, and LLC misses on the cores that my experiment runs on.

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

And there you have it! When adding a new background program, just call it somewhere before the experiment and save its PID. Then, after the experiment, kill it.

### 1.2.3 Application Data

The application itself can tell us a lot of interesting information. Never feel intimidated by altering one of your applications!! You're in the business of kernel development now, an open-source user-level application like iperf can't scare you!

When obtaining new data from the application, first make yourself familiar with the application's file output. For example, iperf produces a json file as output if asked. This is where you would want to put your new data into. I discourage from simply popping a few printfs into the source code. This is poor style and error-prone.  

For some examples of this, you can take a look at my [custom iperf](https://github.com/hema2601/iperf). I added NAPI ID reading (quite poorly), and a proper latency measurement infrastructure for the latencies of packets from netstack entrance to application reception. The README has some more information.

So, essentially, if you want to have new data from the application, you need to integrate it into the original file output, because you cannot really generate multiple output files from one application execution (I mean, you *can*, but at what cost?).

If you are now faced with the problem of having one big output file with way too much info, I recommend going into the `scripts/after.sh` script and have the script make copies of your one big file that you can then name differently and have the Raw Data Converter convert separately. Like here:
```BASH
cat $current_path/iperf.json > $current_path/data/$name/iperf_lat.json
```

