<!--
  Copyright 2019, Data61, CSIRO (ABN 41 687 119 230)

  SPDX-License-Identifier: BSD-2-Clause
-->

# CAmkES Timer Tutorial

This tutorial guides you through setting up a sample timer driver component in
CAmkES and using it to delay for 2 seconds. For this tutorial, we will be using
the ZYNQ7000 ARM-based platform. This platform can be simulated via QEMU so it
is not a problem if you do not have access to the actual physical board.

The tutorial also has two parts to it. The first part will teach you how to
manually define hardware details to configure the hardware component and
initialise hardware resources.  The second part will teach you how to use a
CAmkES connector to initialise hardware resources automatically for you.

The solutions to this tutorial primarily uses the method of manually defining
hardware details. The solutions to the second part are also included, albeit
commented out.

## Initialising

```sh
# For instructions about obtaining the tutorial sources see https://docs.sel4.systems/Tutorials/#get-the-code
#
# Follow these instructions to initialise the tutorial
# initialising the build directory with a tutorial exercise
./init --tut hello-camkes-timer
# building the tutorial exercise
cd hello-camkes-timer_build
ninja
```


## Prerequisites

1. [Set up your machine](https://docs.sel4.systems/HostDependencies).
2. [Camkes 2](https://docs.sel4.systems/Tutorials/hello-camkes-2)

## Exercises - Part 1

### TASK 1

Start in `hello-camkes-timer.camkes`.

Instantiate some components. You're already given one component instance
- `client`. You need to instantiate additional components, a timer driver and
a component instance representing the timer hardware itself. Look
in `components/Timer/Timer.camkes` for the definitions of the components.

Once you open the file, you will notice three different components. The `Timer`
and `Timerbase` components represents the timer driver and the timer hardware
respectively. The `TimerDTB` component represents both the timer driver and the
timer hardware. This component is meant to be used with the `seL4DTBHardware`
CAmkES connector to automatically initialise hardware resources. The second
part of the tutorial will go into more detail about the `TimerDTB` component
and the `seL4DTBHardware` connector.

For now, instantiate the `Timer` and `Timerbase` components.

Note the lines `connection seL4RPCCall hello_timer(from client.hello, to
timer.hello);` and `timer.sem_value = 0;` in the `hello-camkes-timer.camkes`
file. They assume that the name of the timer ''driver'' will be `timer`. If you
wish to call your driver something else, you'll have to change these lines.

### TASK 2

Connect the timer driver component (`Timer`) to the timer hardware component
(`Timerbase`). The timer hardware component exposes two interfaces which must
be connected to the timer driver. One of these represents memory-mapped
registers. The other represents an interrupt.

### TASK 3

Configure the timer hardware component instance with device-specific info. The
physical address of the timer's memory-mapped registers, and its IRQ number
must both be configured.

### TASK 4

Now open `components/Timer/src/timer.c`.

We'll start by completing the `irq_handle` function, which is called in
response to each timer interrupt. Note the name of this function. It follows
the naming convention `<interface>_handle`, where `<interface>` is the name of
an IRQ interface connected with `seL4HardwareInterrupt`. When an interrupt is
received on the interface `<interface>`, the function `<interface>_handle` will
be called.

The implementation of the timer driver is located inside a different folder and
can be found in the `projects/sel4-tutorials/zynq_timer_driver` folder from the
root of the projects directory, i.e. where the `.repo` folder can be found and
where the initial `repo init` command was executed.

This task is to call the `timer_handle_irq` function from the supply driver to
inform the driver that an interrupt has occurred.

### TASK 5

Stop the timer from running. The `timer_stop` function will be helpful here.

### TASK 6

The interrupt now needs to be acknowledged.

CAmkES generates the seL4-specific code for ack-ing an interrupt and provides a
function `<interface>_acknowldege` for IRQ interfaces (specifically those
connected with `seL4HardwareInterrupt`).

### TASK 7

Now we'll complete `hello__init` - a function which is called once
before the component's interfaces start running.

We need to initialise a handle to the timer driver for this device, and store a
handle to the driver in the global variable `timer_drv`.

### TASK 8

After initialising the timer, we now need to start the timer. Do so by calling
`timer_start` and passing the handle to the driver.

### TASK 9

Note that this task is to understand the existing code. You won't have
to modify anything for this task.

Implement the `timer_inf` RPC interface. This interface is defined in
`interfaces/timer.camkes`, and contains a single method, `sleep`, which
should return after a given number of seconds. in
`components/Timer/Timer.camkes`, we can see that the `timer_inf` interface
exposed by the `Timer` component is called `hello`. Thus, the function we
need to implement is called `hello_sleep`.

### TASK 10

Tell the timer to interrupt after the given number of seconds. The
`timer_set_timeout` function from the included driver will help. Note that it
expects its time argument to be given in nanoseconds.

Note the existing code in `hello_sleep`. It waits on a binary semaphore.
`irq_handle` will be called on another thread when the timer interrupt occurs,
and that function will post to the binary semaphore, unblocking us and allowing
the function to return after the delay.

Expect the following output with a 2 second delay between the last 2
lines:
```
Starting the client
------Sleep for 2 seconds------
After the client: wakeup
```

## Exercises - Part 2

Now that you've learnt how to manually define the hardware details of a
hardware component to initialise hardware resources, this part of the tutorial
will teach you how to use the `seL4DTBHardware` connector to do that
automatically.

The connector requires a devicetree blob which describes an ARM platform.
Additionally, the blob's interrupt fields also need to follow the same format
of the ARM GIC v1 and v2. There are devicetree source files bundled with the
kernel, look in the `tools/dts/` folder of the kernel sources. If a suitable
devicetree blob is not available for your platform, then do not proceed with
the tutorial.

### TASK 1

Navigate to the `hello-camkes-timer.camkes` file.

Remove the `Timerbase` and `Timer` component instantiations and instantiate a
`TimerDTB` component instead. Also change the `connection seL4RPCCall
hello_timer(from client.hello, to timer.hello);` and `timer.sem_value = 0;`
lines if necessary.

### TASK 2

Remove the `seL4HardwareMMIO` and `seL4HardwareInterrupt` connections. Connect
the two interfaces inside the `TimerDTB` component with the `seL4DTBHardware`
connector.

### TASK 3

Before opening `components/Timer/Timer.camkes`, remove the `Timerbase` settings
inside the configurations block.

Configure the `TimerDTB` component to pass in the correct DTB path to a timer
to the connector and also initialise the interrupt resources for the timer.
This will allow the connector to read a device node from the devicetree blob
and grab the necessary data to initialise hardware resources. More
specifically, it reads the registers field and optionally the interrupts field
to allocate memory and interrupts.

### TASK 4

Move to `components/TimerDTB/src/timerdtb.c`.

Similar to part one, we'll start with the `tmr_irq_handle` function. This
function is called in response to a timer interrupt. The name of the function
has a special meaning and the meaning is unique to the `seL4DTBHardware`
connector.

The IRQ handling functions of the connector follows the naming convention
`<to_interface>_irq_handle`, where `<to_interface>` is the name of the
interface of the 'to' end in an instance of a `seL4DTBHardware` connection.
Also notice that it takes a `ps_irq_t *` type. This is because, for a given
device, there may be multiple interrupts associated with the device. A
`ps_irq_t` struct is given to the IRQ handling function and it contains
information about the interrupt, allowing the handler to differentiate between
the numerous interrupts of a device.

Likewise with part one, the implementation of the timer driver is in the
included driver in `timer_driver` and the task here is to call
`timer_handle_irq`.

### TASK 5

The timer needs to be stopped, the task here is the same as part one's task 5.

### TASK 6

Again, the interrupt now has to be acknowledged.

For the `seL4DTBHardware` connector, CAmkES also generates and provides a
function to acknowledge interrupts. This function follows a similar naming
convention to the IRQ handler above, `<to_interface>_irq_acknowledge` also
takes in a `ps_irq_t *` argument. Similarly, the `ps_irq_t *` argument helps
CAmkES to differentiate between the possibly many interrupts of a device that
you wish to acknowledge.

### TASK 7 - 10

Task 7 to 10 are the exact same as the tasks in part one.

You should also expect the same output as the first part.


---
## Getting help
Stuck? See the resources below.
* [FAQ](https://docs.sel4.systems/FrequentlyAskedQuestions)
* [seL4 Manual](http://sel4.systems/Info/Docs/seL4-manual-latest.pdf)
* [Debugging guide](https://docs.sel4.systems/DebuggingGuide.html)
* [seL4 Discourse forum](https://sel4.discourse.group)
* [Developer's mailing list](https://lists.sel4.systems/postorius/lists/devel.sel4.systems/)
* [Mattermost Channel](https://mattermost.trustworthy.systems/sel4-external/)
