---
title: Running and Using the Web Server
draft: false
tags:
---
This is a short introduction to how to properly use the webserver. It's nothing complicated.

## Execute the Web Server

You could probably use different web servers, but I have been using a minimal python web server for all my experiments.

**First: Execute the Web Server**

Navigate into the main git repository. This is the place where the `index.html` file and the `data/` folder are. It might look something like this:
```bash
cd ~/Custom_Packet_Steering
```

Then execute the web server. The number at the end defines the web server's port.

```bash
python3 -m http.server 8090
```
Now the webserver is running!

**Second: Access the Web Server**

After launching your webserver, go to your browser of choice and type in your server's IP address followed by the given port number.

`<your server ip address>:8090`

If you connect properly, it will take you to a beautiful website looking like this:
![[Pasted image 20250829160954.png]]


**Third: Get Your Data**

At the top of the website on each half you will see a text field. Into this field, you can type the name of one of your meta-experiment folders. The data in it will be able to be visualized if: 
1. The entered name/path is a valid path that can be accessed from the `data/` directory
2. The entered name/path points to a folder that has a `summaries` directory located within

As an example, in my `data/` directory, I have a folder for my `Baseline_RSS` experiment.
![[Pasted image 20250829162657.png]]
In that directory, there is a `summaries` directory.
![[Pasted image 20250829162734.png]]

So if I enter `Baseline_RSS` into the web server field and hit "Get Value!", the graphs will be loaded.

The graphs won't pop up immediately. Every grey field you see under the text box is a collapsible graph. If you want to see a specific graph, you have to click it. Then, the graph will appear
![[Pasted image 20250829162911.png]]

Don't worry if you don't have the data for every graph in the webserver. If a certain graph cannot be rendered, it will simply not show up.
![[Pasted image 20250829163037.png]]

The two halves work identically, so that you can compare data side by side.

>[!hint]- What if you don't know the name of your meta-experiment directory?
>This happens to me a lot. I ran a lot of experiments. You can obviously check with `ls` on your server, but if you want to check directly in the webserver, just access the `data/` directory. Change the URL to `<your server ip address>:8090/data` and you have direct access. You can even browse your data from here.
>![[Pasted image 20250829163308.png]]

