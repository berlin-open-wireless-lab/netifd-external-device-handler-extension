# How It Works

External device handlers are processes independent from netifd. They use the ubus IPC system to communicate with netifd. An external device handler has almost the same interface as the the hard-coded device handlers built into netifd. It consists of the following actions:

 * **create**
 * **reload**
 * **dump_info**
 * **dump_stats**
 * **free** (as in *delete*)

and in some cases these *hotplug operations* (more on those later):

 * **prepare**
 * **add**
 * **remove**

These are a direct mapping to functions of the same name in netifd's `device_type` structure ([see here](http://git.openwrt.org/?p=project/netifd.git;a=blob;f=device.h;h=e13e43509bee8334330a4a32d2f7f11c6bbc556c;hb=HEAD#l61))

External device handlers expose these operations via ubus methods.
Netifd also suscribes to the external device handler's ubus object to receive notifications asynchronously when an operation terminates.

## Communication between netifd and the external device handler

Netifd and the external device handlers communicate using the ubus IPC system. When netifd creates a device handler stub, it will attempt to subscribe to the external device handler's ubus object.
Although, it will wait for that object to appear before, it is strongly recommended to ensure that the external device handler is started *before* netifd. A good way to achieve this is to use a procd init script with `START` set to a value lower than that of netifd.
If the external device handler crashes, netifd will suspend all related operations and wait for it to re-appear. If the external device handler is stateful, it is up to the author of the handler to recover from the crash.

The following describes the process of device creation with external device handlers. All of the asynchronous operations work similarly using the *pending* flag and timeouts:

1. When netifd first starts, it initializes device handler stubs that act as relays to external device handlers and stores them in a list.

2. Next, the network interface configuration file `/etc/config/network` is parsed. When netifd encounters an interface section, it uses the string given in the `type` field to look up the appropriate device handler structure and calls its `create` function.

3. If the structure is a relay stub generated at start-up, this structure generates a representation of the device, stores it and marks it *pending*. It then simply forwards the configuration blob it is given to the external device handler via ubus. Finally, it starts a timer with a callback function to check back on the device after some time has gone by.

4. The external device handler receives the configuration as arguments to its `create` ubus method and creates the device accordingly. This happens asynchronously.

5. There are two possible scenarios for the next step at the device handler:
 1. When the external device handler has created the device, it fires a *create*-notification.
 2. If the external device handler fails to create the device, it will return a ubus status code unequal to 0. Optionally, it can provide an error message to netifd for it to write to the log.

6. Depending on what happened in step 5, one of the following two things happens at netifd:

  1. If the device was created successfully, netifd reacts to the notification by marking the local device representation *synchronized* and stopping the timer it started in 3. 
  2. If device creation failed and the *create*-notification is never fired, netifd's timer will eventually expire, triggering a re-try of the operation. Since netifd cannot know (at least right now), if device creation failed due some permanent error (like invalid configuration) or a temporary error (such as timing-related problems), it will repeat the action up to 3 times before logging the failure and giving up.

**dump_info** and **dump_stats** are the two exemptions from this process. As they have to work synchronously, they do not involve the *pending* flag or timers and do not require signalling via the subscription mechanism. See the tutorial on writing the device handler's ubus interface for details.
