---
title: Adding New Visualizations
draft: false
tags:
---
All visualizations are done using vega lite and are displayed through the webserver. If you want to add a visualization, you have to:
1. Create your visualization in vega lite
2. Create and include your visualization as a file into the webserver
3. Embed it into the webserver's html

Once you're familiar with vega lite, the rest is very simple

## 2.1 Create your visualization

As I said, you should create your visualization using [vega lite](https://vega.github.io/editor/#/examples/vega-lite/bar)
Its a super powerful tool that is very handy with automization.

When creating your visualization in the browser, you will have to get your raw json data and paste it in to work on it. Make sure its not too big or your vega lite will lag or even crash.

To get the raw data, you can just run your webserver and instead of accessing the index, you can access the `./data` subdirectory. 

![[Pasted image 20250807140010.png|700]]

Then you just click to find the file you want to visualize and copy its json into vega.

It might look like this: (the raw json was copied into the 'values' field)
![[Pasted image 20250807143120.png|1200]]

## 2.2 Create and include your visualization as a file

To integrate your vis, you need to include it in the webserver.
I save every visualization in its own file within `web/components`
![[Pasted image 20250807143355.png|1000]]
The file for the vis from above looks like this:
```java script
var netstat = {
		"$schema": "https://vega.github.io/schema/vega-lite/v6.json",
		"description": "A simple bar chart with embedded data.",
		"data": {"name":"myData"},
		"repeat":["TCPOFOQueue", "TCPHPHits"],
		"spec":{
		"mark": {"type":"bar", "tooltip":true},
		"encoding": {
				"x": {"field": "Conns", "type": "nominal","sort":[], "axis": {"labelAngle": 0}},
				"xOffset":{"field":"Exp"},
				"y": {"field": {"repeat":"repeat"}, "type": "quantitative", "aggregate":"mean"},
				"color": {"field":"Exp"}
		}
		}
}
```

There are two things to note here:
1. The entire vis is assigned into a variable. This variable will be how you interact with the graph in the webserver
2. In the `data` field, instead of the raw values, we say that the name of the data is `myData`. In my implementation, this is mandatory because the webserver will read any data into `myData`. This is obviously implementation-dependent.

Just save your new vis in the same format into a file of your choice.

Then include the source into the webserver `index.html` alongsides the other graphs.

![[Pasted image 20250807144156.png|800]]

## 2.3 Embed the vis into the webserver

Now that the vis is stored in a file and included in the webserver, we need to make the space where it will be displayed.
This is obviously possible in multiple different ways. Here, I will just introduce how *I* added visualizations into *my* webserver implementation.
Feel free to get creative if you want.

As a reminder, the webserver looks like this:
![[Pasted image 20250807144604.png|1800]]

It has two columns to represent different results next to each other. None of this is automated, but completely hard-coded (*shameful* I know...).

Each graph has therefore two sections in the html:
Left Column:

![[Pasted image 20250807144801.png|700]]

Right Column:

![[Pasted image 20250807144830.png|700]]

The naming of the inner div is up to you, but it is necessary that the second one has the same id as the first one attached with a 2. The code logic depends on it...

Then, lastly, you need to add your vis into the `loadAllGraphs` function. 
![[Pasted image 20250807145227.png|1200]]
This function dispatches the `fetchJSONData` function, which takes 3 arguments: 
1. A path to the json file you are visualizing
2. A list of the div-ids that visualizations will be loaded into
3. A list of the variables that hold the visualizations that will be loaded into the divs

Lets say you are adding a visualization for some `new_data` file.

These will be all the changes necessary to fully integrate your new visualization into the webserver! Easy, right~
```java script
//========== web/components/new_data_graph.js ========== //

//The actual vis

var new_data ={

	/* Your amazing vega lite visualization */

}

//========== index.html ================================ //

		<script src="web/components/new_data_graph.js"></script>
[...]

//Left Column 
<button type="button" class="collapsible">New Data Graph</button>
		<div class="content">
			<div id="New_Data" style="overflow-x:scroll;" class="col-md-12"></div>
		</div>

[...]
//Right Column 
<button type="button" class="collapsible">New Data Graph</button>
		<div class="content">
			<div id="New_Data2" style="overflow-x:scroll;" class="col-md-12"></div>
		</div>

[...]
function loadAllGraphs(path, postfix){
	[...]
	fetchJSONData(path.concat("summary_new_data.json"), ['#New_Data'.concat(postfix)], [new_data])
}
```

Also, the reason why the `fetchJSONData` function takes lists as arguments is so that you can load different graphs on the same data file without having to read the json again.

That means that if you have two different visualizations for your data..

**Do not do this**:
```java script
	fetchJSONData(path.concat("summary_new_data.json"), ['#New_Data'.concat(postfix)], [new_data])
	fetchJSONData(path.concat("summary_new_data.json"), ['#New_Data_Plus'.concat(postfix)], [])
```
**Do THIS:**
```java script
	fetchJSONData(path.concat("summary_new_data.json"), ['#New_Data'.concat(postfix), '#New_Data_Plus'.concat(postfix)], [new_data, new_data_plus])
```

And that's all on adding your own visualization to the webserver!