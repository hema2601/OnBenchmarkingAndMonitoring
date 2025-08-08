---
title: Packet Reception and Packet Steering in the netdevice Subsystem of the Linux Kernel
draft: true
tags:
---
 


# An Overview

The purpose of this document is to describe in detail what happens after packets have physically arrived on the NIC until the packets are passed to the network stack.
What makes this part of packet processing interesting is that it lays the foundation for fast and efficient protocol processing. Protocols such as IP and TCP and others operating at
the higher layers are not as easily optimized because they represent a contract with the other communicating party. They have to strictly adhere to preset rules and therefore have by definition less wiggle-room for developers to go crazy with their optimizations.
The netdevice subsystem only has to care about adhering to the semantics of the driver (which is taken care of by the driver code - which is also readily modifiable) and passing interpretable packets to the upper layers. This means that changes are contained to one's own system and optimization is therefore much easier.
Most major performance improvements (that I am aware of at least) have been made in this subsystem. Starting with the introduction of NAPI that reduced the amount of interrupts, over improvements such as GRO that reduces the number of packets the network stack needs to deal with, and skb pooling, which reduces the memory de-/allocation overhead per packet, all the way to the introduction of XDP, which provides a framework for developers to perform some high-level operations on a raw packet, therefore entirely circumventing the high-overhead processing steps of the netstack when possible.
I hope that with this document, I can properly convey what I find so fascinating and enticing about this subsystem. Here, the world truly is your oyster. 
I will try my best to demystify the inner workings of this subsystem, so that you, too, can realize that the kernel is in fact not an untouchable monolith, but a malleable piece of software, where *everything* is done *somewhere* in some piece of code that is just waiting for you to find it and change it.

# The NIC-OS Communication Structure

## At a high level

## At the code level

This is the part where my earlier claim of "everything is done somewhere in some piece of code" reaches its limit. When it comes to device-OS communication, there is inevitably some part that has to run inside the hardware, at least to establish an initial connection when a new device is connected to the system etc. However, the good thing is that since any hardware device needs to be able to function with a wide variety of systems, the steps taken by the hardware are well-defined, so even though we cannot check them explicitly in any piece of code, it is enough to have an abstract idea of what the device is supposed to do. Also, for the same reason, a surprising amount of hardware-near setup tasks are done entirely in software. 
So lets find the main communication points - interrupts, DMA memory, and descriptors - in the kernel code! Specifically in the driver.

 Every device relies on a different driver, so in order to look at any code, we need to decide on one. For this, we will look at Intel's ice driver (the version included in the Linux kernel). 

Since we are interested in the setup routine of the driver, we need to go to the very start of initialization. If this were a normal program, we'd look for the main function. However, drivers are not normal programs, they are *kernel modules*. Kernel modules are pieces of code that can be inserted into the kernel. Instead of a main function, they have to define a function for loading and unloading itself into the kernel. This is a great starting point when analyzing driver code from the very bottom up.

The functions used to define those entry and exit points are `module_init()` and `module_exit()`
![[Pasted image 20250804141940.png|600]]
So here we have it: The beginning of the ice driver! `ice_module_init`
A lot of things are being set up in this function and it is okay to ignore some of those. What we are interested in is the function `pci_register_driver()`.
Why? Because the PCI subsystem (**P**eripheral **C**omponent **I**nterconnect) is in charge of dealing with peripheral devices such as the NIC. When a new device docks on, the PCI subsystem needs to deal with the initial setup - at least as a relay point. Therefore, any information we register with it will be relevant to the driver setup.
So lets take a look at the argument of the function call: `ice_driver`
![[Pasted image 20250804160453.png|500]]
As the name already indicates, this is an incredibly fundamental piece of the driver: Its an interface of function pointers. You encounter these everywhere in the kernel.
What is happening is that the PCI subsystem defined a very clear set of functions it needs to know about so that it can interact with a device. It needs to know how to probe for the device, how to remove it, how to shut it down etc. This is the driver telling the PCI subsystem which *specific* functions it can use. It also provides another important piece of information: the `id_table`. 
![[Pasted image 20250804161159.png|500]]
What we see here is the table of all devices that will want to be matched with the ice driver. For example, `E810C_QSFP` is the name of one of the Intel NICs used in our servers.
This table together with the other functions will be registered to the PCI subsystem. 

From here, we have a vague idea of how the setup of a new device is most likely going to proceed:

```c
/* THIS IS PSEUDO CODE */

// List of all drivers registered with the PCI subsystem
struct pci_driver *registered_drivers;

// Let's assume a new device advertises itself using its device_id to the PCI subsystem
int on_new_device(int device_id)
{
	// We check every driver
	for (int i = 0; i < num_registered_drivers; i++) {
		// If the advertised id is in the drivers registered list
		if (check_id_table(registered_drivers[i].id_table, device_id) {
			// We call the registered probing function 
			return registered_drivers.probe()
		}
	}
}
```

This is the beauty of these function pointer interfaces. The PCI subsystem can be written in a completely general way without any driver knowledge. Still it can easily call into the driver code, because the function pointers were provided, and can thereby communicate with any device - provided the driver has been installed.

We do not need to get further into the details of the PCI subsystem at this point. Just keep in mind that it is the subsystem that deals with the initial contact to the peripheral devices and relays any communication between device, memory, and the host system.

Let's take a look then at `ice_probe`, the *actual* function that is called when we set up a new device. So far, `ice_module_init` simply sets up the driver. Keep in mind that you can install a driver on your system, even when no device needing said driver is currently connected. 
Therefore, the real setup starts in `ice_probe`. 
![[Pasted image 20250804163411.png|500]]
The actual `ice_probe` function is too long to capture, but just from the comments, we can see we are in the right place.

## Pointers for Individual Code Exploration

`ice_module_init` - driver initialization
`ice_probe` - device setup
`ice_init_interrupt_scheme` - interrupt setup

# The Way of the Interrupt

# NAPI

## What is NAPI
## Where is NAPI?

