netstar
========

This repository contains the netstar framework for building asynchronous network functions.

Installation
------------

* Currently, we have only tested netstar on Ubuntu16.04. So please use Ubuntu16.04 to install netstar.

* Please note, to build netstar, it is recommended that your machine has over 16GB memory. Otherwise the build process may fail.

* Clone this repository with `git clone https://github.com/netstar-project/netstar.git`

* `cd` to the `netstar` folder.

* Update three submodules including dpdk, fmt and c-ares. The three submodules are located at "https://github.com/duanjp8617/dpdk.git", "https://github.com/duanjp8617/fmt.git" and "https://github.com/duanjp8617/c-ares.git" respectively. You can use `git submodule update --init` to update the three submodules.

* Install required dependencies using `sudo ./install-dependencies.sh`.

* Configure netstar with `./configure.py --enable-dpdk --compiler=g++-5 --mode=release`

* Build netstar with `ninja`.

Structure of Source
--------------------

This repository is created based on seastar. The core of netstar is located in the `netstar/` folder. Several netstar applications are located in the `test/netstar/` folder.

* `netstar/asyncflow`: Contain the implementation of async-flow interface. The `netstar/asyncflow/sd_async_flow.hh` contains the implementation of the async-flow manager and the async-flow object.

* `netstar/hookpoint`: Contain the implementation of several hook points.

* `netstar/mica`: Contain the client interface for mica server.

* `netstar/preprocessor`: Contain the implementation of several preprocessors.

* `netstar/stack`: Contain an interface for using the Seastar TCP/IP stack within netstar.

* `port.hh` and `port_manager.hh`: The interface for creating a port abstraction for receiving and sending packets.

* `rte_packet.hh`: A wrapper around the DPDK `rte_mbuf`. This wrapper avoids the memory copy of `rte_mbuf`.

* `shard_container.hh`: Contains the definition of some utility functions.

* Note that we also add some patches to the original seastar source code. You can find out the patch that we created by searching for the `patch by djp` key word in the `net/` folder.

The Seastar Documentation and C++11/14 Tutorial
------------------------------------------------

netstar extensively uses seastar, especially the future/promise implementation provided by seastar. The following materials can be referred to help you understand how netstar works.

* A comprehensive tutorial of seastar, which is available at: "https://github.com/scylladb/seastar/blob/master/doc/tutorial.md"

* netstar and seastar are both implemented using C++11/14. A good tutorial about C+11/14 features can be found in the famous book `Effective Modern C++` by Scott Meyers.

A Tutorial of NetStar
----------------------

We give a tutorial based on a simple packet forwarder implemented using netstar. The source code is in `tests/netstar/simple_forwarder.cc`.

1. The `simple_forwarder.cc` implements a simple packet forwarder, which receives the packet from one port, and send the packet out from another port.

2. Initialize the basic seastar runtime system. This is done with line 132 `return app.run_deprecated(ac, av, [&app] {`.

3. Use the port_manager to add two ports for sending and receiving packets.
```cpp
port_manager::get().add_port(opts, 0, port_type::standard).then([&opts]{
    return port_manager::get().add_port(opts, 1, port_type::standard);
})
```
Line 159-161 add two ports. Port 0 (the first added port) is used to receive packets. Port 1 (the second added port) is used to send packets.

4. Add two hookpoints to each port with 161-165.
```cpp
.then([]{
    return hook_manager::get().add_hook_point(hook_type::sd_async_flow, 0);
}).then([]{
    return hook_manager::get().add_hook_point(hook_type::sd_async_flow, 1);
})
```

5. Start the forwarder in line 166.
```cpp
.then([]{
    return forwarders.start();
})
```
The definition of the forwarders is in line 42-43. forwarders is a special template class `distributed<forwarder>`. The `distributed` is a utility provided seastar, which is used to create one instance of the `forwarder` class on each of the program thread.

By default, after initialization.
