---
title: Summarizer
draft: false
tags:
---
The summarizer is a python script called `merger.py`. Its task is to take all the individual experiment directories that were created and pool their data files into summarized versions.

Keep in mind that the summarizer does not calculate any averages etc. It just accumulates the raw data without any loss of detail. The averages that are displayed in the visualizations are all calculated from the raw data using the vega lite visualization grammar.

Lets assume the following data:

```json
// data/RSS_1_1/iperf.json
[
{"Gbps":30}
]
//-----------------

// data/RSS_1_2/iperf.json
[
{"Gbps":32}
]
//-----------------

// data/RPS_1_1/iperf.json
[
{"Gbps":20}
]
//-----------------

// data/RPS_1_2/iperf.json
[
{"Gbps":20}
]
//-----------------
```

This data after running the experiments will be located in separate files - as indicated by the comments. We want to have all this data in one file for easy access and visualization. After the summarizer runs, the data will look like this:

```json
// data/summary_iperf.json
[
	{
		"Gbps":30,
		"Exp":"RSS",
		"Conns":1,
		"Rep":1
	},
	{
		"Gbps":32,
		"Exp":"RSS",
		"Conns":1,
		"Rep":2
	},
	{
		"Gbps":20,
		"Exp":"RPS",
		"Conns":1,
		"Rep":1
	},
	{
		"Gbps":20,
		"Exp":"RPS",
		"Conns":1,
		"Rep":1
	},
]
```

The data that was previously encoded in the directory names was put into the json file itself.

The Summarizer is much more statically coded than necessary, so in order to extend this infrastructure, knowing how it finds its files of interest is important.

The Summarizer looks for its files in the `/data/` subdirectory.
As its fourth argument, it receives a meta-experiment directory name. If no fourth argument is given, it just uses `.` as its meta-experiment directory.
```python
if len(sys.argv) > 4:
    base_dir = sys.argv[4]
else:
    base_dir = "."
```

As mandatory arguments, it also gets the number of reps, the number of connection increases, and whether the connections increase exponentially or not. So it gets very similar arguments to the [[Experiment Iterator]].
It uses this information to compile a list of sub-experiments it expects to find in the meta-experiment directory. Those are stored in an array called `directory`.

```python
directory = []

exp = 1
tmp = []
for i in range(conns_pow_of_2):
    for base in bases:
        if exponential == 1:
            tmp.append(base+"_"+str(exp))
        else:
            tmp.append(base+"_"+str(i+1))
    exp *= 2

for i in range(reps):
    for curr_dir in tmp:
        directory.append(curr_dir+"_"+str(i+1))
```

Note how the code loops over the `base` array to generate the files in the directory. The `base` array contains any possible experiment name given to a single experiment in the [[Setup Iterator]]. This is a terrible design and only worked well when I was only running experiments on RSS, RPS, RFS, or IAPS. Once things got more complicated, the array grew bigger and now has way too many entries to include it here in the text. If you change anything about the Summarizer, start here.

After generating the `directory` array, the Summarizer creates the `summaries` directory in the main directory (in my case `/home/hema/Custom_Packet_Steering`).
This directory will hold all the summarized data files.

Then, the summarizing part begins. The Summarizer loops over an array called `files`, which holds the names of any data file that could be found in the sub-experiment directories. For every such file, it creates a summary file.

```python
current_path="/home/hema/testing_infrastructure/"

os.mkdir(current_path + "summaries")


for f in files:
    
    new_dict = list()
    
    # Create new file
    file_name = "summary_"+f


    with open(current_path + "summaries/"+file_name, "w") as file:
	#[...]
```

Then it loops over all possible experiments we saved in `directory` and checks whether a path to the current data file exists. If it doesn't, the Summarizer skips it.

```python
for exp in directory:
            if os.path.isfile(current_path + "data/" + base_dir + "/" + exp+"/"+f) is False:
                continue
```

Otherwise, we open the file and load its json contents. Then we iterate over every single element and add the `Exp`, `Rep`, and `Conns` items to it. Then, the newly-augmented element is saved into a temporary json dictionary, which is dumped into the summary file at the very end.

```python
 with open(current_path + "data/" + base_dir + "/"+exp+"/"+f) as json_file:
                d = json.load(json_file)
                for elem in d:
                    elem["Exp"]=exp.split("_")[0]
                    elem["Conns"]=exp.split("_")[1]
                    elem["Rep"]=exp.split("_")[2]
                    new_dict.append(elem)

        json.dump(new_dict, file, indent=0)
```

The Summarizer does a very simple job, but is incredibly convoluted due to how it finds its files. Its a great example of why it is worth it to spend some time on writing dynamic code from the beginning, even though it is tempting to just hard-code everything to the format that you are currently using. 

>[!check]- How the Summarizer *should* find its files
>Ideally, the Summarizer would just be given a meta-experiment directory name and automatically go over every sub-experiment folder within, without actively generating their names. This would get rid of the atrocious `directory` array and give the user more freedom when it comes to naming their experiments. Maybe it will do an initial pass over all directories to find which data files are present, so it would build its own `files` array, without the user having to hardcode it. Then, on a second pass, it could just do what the Summarizer is doing now: For every data file, go through all directories and compile them into a summary file. Like this, the Summarizer would work much more dynamically and would be less error-prone.

Another common issue with the Summarizer is the `summaries` directory. Note how it generates the directory in the main directory and not in the meta-experiment directory where it belongs. This is a remnant of when the Summarizer was just used as a command-line tool to summarize my latest experiment, rather than being a small part of my bigger infrastructure. The `summaries` directory is usually moved to the meta-experiment folder by the [[Setup Iterator]] through a helper function from `my_lib.sh`. Just be aware that in case of a failed execution, you need to clean up the `summaries` folder by yourself. Otherwise, the Summarizer will fail on the next execution, because the `summaries` folder already exists.