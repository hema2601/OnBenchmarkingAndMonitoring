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
