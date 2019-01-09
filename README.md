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
The definition of the forwarders is in line 42-43. forwarders is a special template class `distributed<forwarder>`.

By default, after a seastar program is initialized, the runtime creates one thread for each available CPU core and pins the thread to the corresponding CPU core. Different thread shares no information with each other. THe `distributed` class is a utility template class provided by seastar, which is used to create one instance of the `forwarder` class on each of the thread. The `forwarder` class contains the implementation of implements the core logic of the packet forwarder.

6. Line 167-171 tells the hookpoint to start to work.
```cpp
.then([]{
    return hook_manager::get().invoke_on_all(0, &hook::check_and_start);
}).then([]{
    return hook_manager::get().invoke_on_all(1, &hook::check_and_start);
})
```

7. Line 171-173 tells the forwarder to start to work.
```cpp
.then([]{
    return forwarders.invoke_on_all(&forwarder::run_udp_manager, 1);
})
```

8. Line 173-175 collects data from different `forwarder` instances running on different threads and prints the data on the terminal.

9. The details of the `forwarder` class below are covered below.

10. The constructor of the `forwarder` (line 51-63) configures the `forwarder` instance. It basically tells the hookpoint, which async-flow manager should it send packets to and receive packet from.
```cpp
forwarder() {
    hook_manager::get().hOok(0).send_to_sdaf_manager(_tcp_forward);
    hook_manager::get().hOok(0).receive_from_sdaf_manager(_tcp_reverse);

    hook_manager::get().hOok(1).receive_from_sdaf_manager(_tcp_forward);
    hook_manager::get().hOok(1).send_to_sdaf_manager(_tcp_reverse);

    hook_manager::get().hOok(0).send_to_sdaf_manager(_udp_forward);
    hook_manager::get().hOok(0).receive_from_sdaf_manager(_udp_reverse);

    hook_manager::get().hOok(1).receive_from_sdaf_manager(_udp_forward);
    hook_manager::get().hOok(1).send_to_sdaf_manager(_udp_reverse);
}
```

Here, 
