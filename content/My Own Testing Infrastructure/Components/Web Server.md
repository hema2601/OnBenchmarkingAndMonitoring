---
title: Web Server
draft: false
tags:
---

Finally, the web server.

The web server consists of one `index.html` file and the visualizations that it displays are saved under `web/components/`.

Now what is the purpose of the webserver?
Its purpose is to automatically visualize my experiment data from a GUI-less server onto my main machine. I don't have to do any copying, I dont have to put data into Excel, I dont have to actively do anything with the data. After running my experiments, I just run the webserver on my server, access it from my main machine, and give it the name of my experiment suite. It will automatically generate the graphs. A short tutorial on the execution can be found [[Running and Using the Web Server|here]].

In this document, I want to touch down a bit on the internal workings of the web server. Since it is a very simplistic piece of software, we can divide it into two parts:
1. The HTML
2. The JavaScript Logic

## The HTML

The website consists of two columns that are virtually identical, so I will only be describing the structure of a single column.

One column has the text field with the "Get Value!" button, followed by a number of buttons that toggle the divs containing the graphs.

There really doesn't seem to be much else to explain about the html structure...

## The JavaScript Logic

This part will be describing the process of how the the graphs are generated after clicking the "Get Value!" button.

### First: Clicking the Button

The button is connected to the `getValue` function. Based on whether the button of the first or second column is clicked, the `div_num` 1 or 2 will be passed. Then the input directory name is taken from the text field, the heading is updated, and a target path is compiled.

That path has the following form: `data/<text_field_value>/summaries/`

```javascript
function getValue(div_num) {
	// Get the input element by its ID
	if(div_num == 1){

		let inputField = document.getElementById("myInput");

		// Get the value of the input field
		let value = inputField.value;

		document.getElementById("datadir").innerHTML = value;

		loadAllGraphs("data/".concat(value).concat("/summaries/"), "")

	}else{
		let inputField = document.getElementById("myInput2");

		// Get the value of the input field
		let value = inputField.value;

		document.getElementById("datadir2").innerHTML = value;

		loadAllGraphs("data/".concat(value).concat("/summaries/"), "2")
	}
}

```

### Second: `loadAllGraphs`

The `loadAllGraphs` function just dispatches the `fetchJSONData` function for each data file that is supposed to be visualized. It uses the path given as argument to combine it with each data file to be visualized. It also uses the postfix to differentiate between whether the graphs should be loaded on the right or left column.

So the arguments to the `fetchJSONData` calls are:
1. The relative path to the data file
2. A list of div IDs where a viz based on the given data file should be loaded.
3. A list of vega lite graph objects (these are defined in their respective files in `web/components`)

```javascript

function loadAllGraphs(path, postfix){
	

	fetchJSONData(path.concat("summary_iperf.json"), ['#Throughput'.concat(postfix), '#ThroughputMM'.concat(postfix)], [Throughput, ThroughputMM])
	fetchJSONData(path.concat("summary_iperf_lat.json"), ['#RX_Lat'.concat(postfix)], [RX_Lat_histo])
	fetchJSONData(path.concat("summary_packet_cnt.json"), ['#Drops'.concat(postfix)], [Drops])
	fetchJSONData(path.concat("summary_softnet.json"), ['#PpIPI'.concat(postfix), '#PpIPI_err'.concat(postfix),'#InputQData'.concat(postfix)], [PpIPI, PpIPI_err, InputQData])
	fetchJSONData(path.concat("summary_pkt_steer.json"), ['#PktSteer'.concat(postfix), '#BC_Stats'.concat(postfix)], [PktSteer, backup_choice])
	fetchJSONData(path.concat("summary_proc_stat.json"), ['#CPUUtil'.concat(postfix)], [CPUUtil])
	fetchJSONData(path.concat("summary_perf.json"), ['#Perf'.concat(postfix)], [perf_graph])
	fetchJSONData(path.concat("summary_perf_stat.json"), ['#CacheMiss'.concat(postfix), '#IPC'.concat(postfix)], [cache_miss, ipc])
	fetchJSONData(path.concat("summary_busy_histo.json"), ['#Busy_Histo'.concat(postfix),'#Busy_Histo_two'.concat(postfix)], [Busy_Histo, Busy_Histo2])
	fetchJSONData(path.concat("summary_pkt_lat_histo.json"), ['#Pkt_Lat'.concat(postfix)], [Pkt_Lat_histo])
	fetchJSONData(path.concat("summary_pkt_size_histo.json"), ['#Pkt_Size'.concat(postfix)], [pkt_size_graph])
	fetchJSONData(path.concat("summary_netstat.json"), ['#Netstat'.concat(postfix)], [netstat])

}
```

## Third: `FetchJSONData`

`FetchJSONData` executes a chain of actions. Let's go through them step-by-step.

The function starts by simply fetching the data from the path that was given.
```javascript
function fetchJSONData(path, divs, graphs) {
	fetch(path)
```

Then, it needs to check the result. If the fetch failed, because the file didn't exist for example, it throws an error. Else it returns the contents of the file as a json object.
```javascript
function fetchJSONData(path, divs, graphs) {
	fetch(path)
		.then((res) => {
			if (!res.ok) {
				throw new Error
				(`HTTP error! Status: ${res.status}`);
			}
			return res.json();
		})
```

Afterwards, we loop over our list of divs and visualization objects to render them one-by-one. The nth entry on the div list is associated with the nth visualization object. 

```javascript
function fetchJSONData(path, divs, graphs) {
	fetch(path)
		.then((res) => {
			if (!res.ok) {
				throw new Error
				(`HTTP error! Status: ${res.status}`);
			}
			return res.json();
		})
		.then((data) =>
			{
				divs.forEach((div, idx) => {

					vegaEmbed(div, graphs[idx], {tooltip: {theme: 'light'}})
					[...]
				}
				);
			})
```

Then, after every visualization is created, we insert the loaded json data into it. This is done with the `insert` function. Each visualization needs to have a named `data` field with 'myData' for this to work. 
```javascript

function fetchJSONData(path, divs, graphs) {
	fetch(path)
		.then((res) => {
			if (!res.ok) {
				throw new Error
				(`HTTP error! Status: ${res.status}`);
			}
			return res.json();
		})
		.then((data) =>
			{
				divs.forEach((div, idx) => {

					vegaEmbed(div, graphs[idx], {tooltip: {theme: 'light'}}).then(res =>
						res.view
						.insert('myData', data
						).resize()
						.run()
					)
				}
				);
			})
```

Last but not least, a little error handling so that the server knows what to do when the data cannot be found.
```javascript

function fetchJSONData(path, divs, graphs) {
	fetch(path)
		.then((res) => {
			if (!res.ok) {
				throw new Error
				(`HTTP error! Status: ${res.status}`);
			}
			return res.json();
		})
		.then((data) =>
			{
				divs.forEach((div, idx) => {

					vegaEmbed(div, graphs[idx], {tooltip: {theme: 'light'}}).then(res =>
						res.view
						.insert('myData', data
						).resize()
						.run()
					)
				}
				);
			})
			.catch((error) =>
				console.error("Unable to fetch data:", error));
}
```

And that's it, that's pretty much the entire logic of the webserver.