---
title: Setting up and Running any Experiments in Your Own Environment
draft: false
tags:
---
You want to run some of my experiments? Amazing! 

Here is what you need to change in the code to adjust it to your environment:


# Get the repository

The repository is available [here](https://github.com/hema2601/testing_infrastructure)

Just do the following to get it to your server:

```bash
git clone https://github.com/hema2601/testing_infrastructure.git
```

# Edit and Run the `configure.sh` script

To make configuration easy, I included a configuration script called `configure.sh`. There are 5 things you will have to write into it.

1. The path of the github repository
2. Your iperf binary
3. Your perf binary
4. your remote client ssh access
5. Your server's network interface IP

For 2 and 3 you can either give a binary name, or a path. For example, when running a custom kernel, it is sometimes hard to actually install `perf` on your system, so you might just locally build it using `make` and then put the path to that local perf into the configuration script.

```bash
# The absolute path to this github repository
CURR_PATH=/home/hema/testing_infrastructure/

# How to access your iperf binary
STANDARD_IPERF_BIN=iperf3

# How to access your perf binary
PERF_BIN=perf

# How to ssh onto your remote node
REMOTE_SSH=user@xxx.xxx.xxx.xxx

# The IP address of this server's target network interface
SERVER_TARGET_IP=30.0.0.3
```

## Some Common Errors When Running the Script

### sed: Couldn't open temporary file .../sed\*: Permission denied....
-> Run as sudo (in this case, all paths have to be absolute, not beginning with `~`)

### sed: ... : Unknown Option to \`s'
-> In your configured values, there might be a special character that is interpreted by `sed`. You need to escape it with a `\`. If you look at the configuration script, you see that I do this with the `/` in file paths. So instead of having a path like `/my/path`, it is written as `\/my\/path`. 

## The script doesn't work?

Do the configuration manually~ Its pretty simple

### github repository path 

Do a grep for all instance of `current_path`
```bash
grep -r --exclude=configure.sh "current_path="
```
Manually replace all those results with your own github repository path.

### iperf binary

Open `run_mini_project.sh` and edit the following part at the top to include your iperf binary of choice:
```bash
IPERF_BIN=iperf
```

### perf binary
Open `run_mini_project.sh` and edit the following part at the top to include your perf binary of choice:
```bash
PERF_BIN=iperf
```

### Remote client
Open `run_mini_project.sh` and edit the following part at the top to include your ssh target:
```bash
remote_client_addr=user@xxx.xxx.xxx.xxx
```

### Server IP
Open `run_mini_project.sh` and edit the following part at the top to include your server's IP:
```bash
server_ip=30.0.0.3
```

# Setup up password-less SSH

In order to fully automate the script, it is important that you can ssh onto your remote node without entering the password.

>[!warning] Do not set this up if you are not in a trusted environment
>This might go without saying, but consider where you set up password-less ssh. Make sure the connection you are setting up is okay to be connected without using passwords aka can everybody that can access your current server be trusted with being able to log in to the remote node without a password?

Since we need to run our experiment with root privileges on the current server, we need to set up a password-less connection between the root account of your server and your user account on the remote node.

Do the following:

1. On your server, log into root
```bash 
sudo su
```

2. Create a key for the root account and press enter on all options to make it password-less
```bash
ssh-keygen -t rsa -b 4096
```

3. Verify that the key was successfully generated:
```bash
ls -al ~/.ssh/id_*.pub
```

4. Copy the key to your remote client
```bash
ssh-copy-id [remote_username]@[remote_ip_address]
```

5. Test whether everything worked by sshing onto the remote server
```bash
ssh [remote_username]@[remote_ip_address]
```

If you have any issues, refer to [this tutorial](https://phoenixnap.com/kb/setup-passwordless-ssh). This is where the original instructions are from.

# Run the experiment

If everything worked, the basic experiment should run out-of-the-box. Try running it like this:

```bash
cd experiment
sudo ./baseline_experiment.sh <your interface name> 2 2
```

This will run a total of 12 iperf experiments. If it finished successfully, you can [[Running and Using the Web Server|execute and visualize your data with the web server]].