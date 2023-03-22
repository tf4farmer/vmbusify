
# VMBusify(vmbusify) solution and specification

## Lead

VMBusify(vmbusify), provide a way for communication and control between physical and virtual machines based on Microsoft VMBus architecture, this name combines 'VMBus' and 'simplify', and suggests that the software simplifies communication and control between the physical and virtual machines.

## Requirements

we have two cases:

1. after virtual machine created, usually, the SDN network is not ready immediately and the virtual machine is not accessible, in such a period before SDN network ready, we need an alternative way to set up the communications between the virtual machines and external network, then we can directly install softwares and perform initialization tasks on the virutal machine without SDN networking, shorten the virtual machine preparation time.

2. when the SDN network encouters some failures, there would be no route from the virtual machines to access external network and also they are unreachable from external network, in such case, we need an alternative way to recover the communications between the virtual machines and external network, to repair the network fault, making the services on virtual machines available for winning a highly continuity, shorten the downtime.

in summary, an alternative way is very necessary for operations, commnunications and controls to virtual machines when SDN network is unavalable, based on Microsoft Hyper-V architecture, VMBus provide us such an alternative way and this project is based on this technology.

## Approaches

for commnucation and control between physical and virtual machines based on VMBus technology, we provide the following three approaches.


### 1. command-execution

command line tool running on physical machine, similar with the PSExec tool, use it to connect to a target virtual machine and execute commands with stdin and stdout being redirected to the physical machine

typical usage:

		vmbusifyexec <VMID> [-it] <program> <args>

### 2. socks-proxy

a bidirectional socks proxy via vmbus as a underlay tunnel between the physical machine and virtual machines that it holds,	processes on the virtual machine can use this proxy to access the external network by the physical machine as the socks proxy server,	processes on the physical machine also can use this proxy to access the tcp services on the virtual machines

### 3. virtual-subnet

networkng for the virtual machines and physical machine, a virutal subnet can be built among the physical machine and all virutal machines that it holds, with the physical machine as the NAT for this subnet connecting to external network, being different from normal network, this virutal subnet use vmbus as the physical layer instead of Ethernet and MAC

## Architecture

in summary, below is the architecture overview illustration

```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

                                                                                                   +------------+
                                                                                                   |            |
                                                                                                   | httpserver <-----------------------------------------+
                                                                                                   |            |                                         |
                                                                                                   +-----^------+                                         |
                                                                                                         |                                                |
                                                                                                         |                                                |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+
|                                                                          |        |                    |                                                |                     |
|                                                                          |        |                    |                                                |                     |
|      +----------+           +---------+      +----------+  +--------+    |        |   +------------+   |   +--------------------+  +-------------+      |                     |
|      | curl ... |           | ping... |      | wget ... |  |  sshd  |    |        |   | ssh client |   |   | vmbusifyexec ping..|  | vmbusifyctl |      |                     |
|      +----+-----+           +----^----+      +-----+----+  +---^----+    |        |   +-------+----+   |   +---+----------------+  +--+----------+      |                     |
|           |                      |                 |           |         |        |           |        |       |                      |                 |                     |
|           |                      +-------+         |           |         |        |           |        |       |   +------------------+                 |                     |
|           |                              |         |           |         |        |           |        |       |   |                                    |                     |
|           |                +--------+----+-----+---v----+------+-+       |        |       +---v----+---+----+--v---v---+--------+                       |                     |
|           |                |        |          |        |        |       |        |       |        |        |          |        |                       |                     |
|           |                | ip     | command  | proxy  | proxy  |       |        |       | proxy  | proxy  | command  | ip     |                       |                     |
|           |          +-----> bearer | executor | client | server |       |        |       | client | server | listener | bearer <----+                  |                     |
|           |          |     |        |          |        |        |       |        |       |        |        |          |        |    |                  |                     |
|           |          |     +--------+----------+--------+--------+       |        |       +--------+--------+----------+--------+    |                  |                     |
|           |          |     |                                     |       |        |       |                                     |    |                  |                     |
|           |          |     |           vmbus tunnel              |       |        |       |            vmbus tunnel             |    |                  |                     |
|           |          |     |                                     |       |        |       |                                     |    |                  |                     |
|           |          |     +-----------------^-------------------+       |        |       +------------------^------------------+    |                  |                     |
|           |          |                       |                           |        |                          |                       |                  |                     |
|           |          |                       |                           |        |                          |                       |                  |                     |
|     +-----v------+   |               +-------v--------+                  |        |                  +-------+--------+              |             +----+-----+               |
|     |            |   |               |                |                  |        |                  |                |              |             |          |               |
|     | tcp-socket |   |               |  vmbus-socket  |                  |        |                  |  vmbus-socket  |              |             |    NAT   |               |
|     |            |   |               |                |                  |        |                  |                |              |             |          |               |
|     +-----+------+   |               +-------^--------+                  |        |                  +-------^--------+              |             +----^-----+               |
|           |          |                       |                           |        |                          |                       |                  |                     |
|           |          |                       |                           |        |                          |                       |                  |                     |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+
|           |          |                       |                           |        |                          |                       |                  |                     |
|           |          |                       |                           |        |                          |                       |                  |                     |
|      +----v----+  +--+-----------+   +-------v--------+                  |        |                  +-------v--------+          +---v---------+   +----v----+                |
|      |   tcp   |  |              |   |                |                  |        |                  |                |          |             |   |   tcp   |                |
|      +---------+  |   vir-ethio  |   |  vmbus-device  <---------------------------------------------->  vmbus-device  |          |  vir-ethio  |   +---------+                |
|      |   ip    |  |              |   |                |                  |        |                  |                |          |             |   |   ip    |                |
|      +------^--+  +--^-----------+   +----------------+                  |        |                  +----------------+          +----------^--+   +---^-----+                |
|             |        |                                                   |        |                                                         |          |                      |
|             |        |                                                   |        |                                                         |          |                      |
|             +--------+                                                   |        |                                                         +----------+                      |
|                                                                          |        |                                                                                           |
|                                                        virtual machine   |        |  physical machine                                                                         |
|                                                                          |        |                                                                                           |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+

```

### vmbus-tunnel

this is the transport layer based on VMBus between physical and virtual machines, for each virtual machine, physical machine could establish one VMBus connection with it and provide TLV (Type-Length-Value) format package switch capability by time division multiplexing. we define the TLV format data transfer unit as vmbus package.

so, for the virtual machine endpoint, it will listen on VMBus and then accept the VMBus connection, only one connection would be kept, new connection would be accepted unless current connection being released.

for the physical machine, it will connect to a virtual machine on demand, also, user could make the physcial machine connect to all virtual machines that it holds and keep the connections available for the full life cycle. typically, physical machine endpoint will keep a set indicating relationship from VMID(virtual machine identity) to the socket of VMBus connection.

virtual machine connection relationship two-tuple:

		{<VMID, socket>}

as the following illustration, physical machine manages multiple connections and only one connection for each virutal machine, for the vmbus-tunnel, we only support sending and receiving messages between the virtual machine and physical machine, dispatching message from one virtual machine to another virtual machine via physical machine by vmbus-tunnel is not support.


```
+-----------------+
|                 |                              <vmid1, sock1>
| virtual machine <--------------------------------+
|                 |                                |
+-----------------+                                |
                                                   |
                                                   |
+-----------------+                                |
|                 |            <vmid3,sock3>       |
| virtual machine <---------------+                |
|                 |               |                |
+-----------------+               |            +---+--------------+
                                  +------------+                  |
                                               | physical machine |
+-----------------+               +------------+                  |
|                 |               |            +---+--------------+
| virtual machine <---------------+                |
|                 |           <vmid4, sock4>       |
+-----------------+                                |
                                                   |
                                                   |
+-----------------+                                |
|                 |                                |
| virtual machine <--------------------------------+
|                 |                              <vmid2, sock2>
+-----------------+

```

multiplexing on each connection between virtual machine and physical machine requires the tunnel as a serialized transmission channel for all TLV format vmbus packages, this makes the data transmission mechanism simple and easy to understand, meanwhile, we provide both synchronized and asynchronized mode for sending data. for the upper layer of vmbus-tunnel illustrated in the graph, such as proxy client/server, command listener/executor and so on, we define them as the application layer, for each entity in the application layer, we define the application layer transfer unit which means the payload of vmbus package as application message. as the illustration below, the hierarchy architecture contains two layers, the infrastructure is vmbus-tunnel for transmitting vmbus-package and the upper layer is application layer for transmitting application-message.

```

+--------------------+         +--------------------+
|                    |         |                    |
|    application     +---------> applicaton-message |
|                    |         |                    |
+--------------------+         +--------------------+
|                    |         |                    |
|    vmbus-tunnel    +--------->    vmbus-package   |
|                    |         |                    |
+--------------------+         +--------------------+

```

below is the illustration that indicating the elements of vmbus package, the 'sig' field is the signature number that represents the head of a package, 'type' indicates that whether the package is a normal request or an acknownlegement for a given request. 'id' is a counter for each package that sending from current endpoint while for an acknownlegement it will be same with its related request. 'message-id' and 'application-entity' decide the message handler for the application message to dispatch, 'length' indicates the bytes of payload which contains the application message.

```

+-----+------+----+------------+--------------------+--------+---------------------------------+
|     |      |    |            |                    |        |                                 |
| sig | type | id | message-id | application-entity | length | payload (application message)   |
|     |      |    |            |                    |        |                                 |
+-----+------+----+------------+--------------------+--------+---------------------------------+

+                                                                                              +
|                                                                                              |
+----------------------------------------------------------------------------------------------+

                                    vmbus-package

```

vmbus-tunnel provide a registry service for them to declare themselves by using their entity names and then, declare all message id they wish to receive and message handler for processing it. the primary key for message is <entity-name, message-id>, for both endpoint, the vmbus-tunnel keeps a set of two-tuple, we call it application message route table.

application message route table two-tuple, the second element of the tuple is a set of triple-tuple that indicating all messages that the entity wish to receive and their message handlers.

		{<self-entity-name, {<message-source-entity-name, message-id, message-handler>}>}

message-id indicates what message they want and message-source-entity-name indicates from which entity they care, if and only if the given message from the given source entity would be targeted to the declared message handler. for each mesage, if there exists more than one handler, then each handler would be triggered serially in order of registration.

### command-execution approach

to to sum up the command exectuion machanism, the vmbusifyexec, command listener, vmbus-tunnel and command executor get together to constitute an application for remote commands execution, such as the Windows internal tool 'PsExec'.

```
                            physcial machine                                                                   virtual machine
+------------------------------------------------------------------------+          +----------------------------------------------------------------------+
+                                                                        +          +                                                                      +

+----------------+          +----------------+          +----------------+          +----------------+         +----------------+         +----------------+
|                |          |                |          |                |          |                |         |                |         |                |
|  vmbusifyexec  |          |  cmd-listener  |          |  vmbus-tunnel  |          |  vmbus-tunnel  |         |  cmd-executor  |         |     process    |
|                |          |                |          |                |          |                |         |                |         |                |
+-------+--------+          +-------+--------+          +-------+--------+          +--------+-------+         +--------+-------+         +-------+--------+
        |                           |                           |                            |                          |                         |
        |       http request        |                           |                            |                          |                         |
        +--------------------------->                           |                            |                          |                         |
        |     exec cmd on vmid      |                           |                            |                          |                         |
        |                           |         call              |                            |                          |                         |
        |                           +--------------------------->                            |                          |                         |
        |                           |     cmd-exec message      |                            |                          |                         |
        |                           |                           +----+                       |                          |                         |
        |                           |                           |    | get sock by vmid      |                          |                         |
        |                           |                           |    |                       |                          |                         |
        |                           |                           <----+                       |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           |        vmbus socket        |                          |                         |
        |                           |                           +---------------------------->                          |                         |
        |                           |                           |       vmbus package        |                          |                         |
        |                           |                           |                            +----+                     |                         |
        |                           |                           |                            |    | dispach message     |                         |
        |                           |                           |                            |    |                     |                         |
        |                           |                           |                            <----+                     |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |        callback          |                         |
        |                           |                           |                            +-------------------------->                         |
        |                           |                           |                            |    cmd-exec message      |                         |
        |                           |                           |                            |                          |      create process     |
        |                           |                           |                            |                          +------------------------->
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          |    process terminated   |
        |                           |                           |                            |                          <-------------------------+
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |          call            |                         |
        |                           |                           |                            <--------------------------+                         |
        |                           |                           |                            |     exec-ret message     |                         |
        |                           |                           |       vmbus socket         |                          |                         |
        |                           |                           <----------------------------+                          |                         |
        |                           |                           |      vmbus package         |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           +----+                       |                          |                         |
        |                           |                           |    | dispatch message      |                          |                         |
        |                           |                           |    |                       |                          |                         |
        |                           |                           <----+                       |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           |        callback           |                            |                          |                         |
        |                           <---------------------------+                            |                          |                         |
        |                           |      exec-ret message     |                            |                          |                         |
        |      http response        |                           |                            |                          |                         |
        <---------------------------+                           |                            |                          |                         |
        |      exec result          |                           |                            |                          |                         |
        +                           +                           +                            +                          +                         +


```

#### vmbusifyexec

this entity is a command line tool for executing command on virutal machines via the physical machine. it only deploys on pysical machine endpoint. usage of this tool is similar to 'PsExec'.

#### command-listener

this entity only deploys on physcial machine endpoint, listen on tcp 127.0.0.1:6543, providing an operation interface for command line tools such as vmbusifyctl and vmbusifyexec. we defined two types of commands, the first class is built-in commands for controlling the vmbus-tunnel, for example, connect to or disconnect from given target virutal machine, the second class is application specified commands, for example, execute ping command on given target virtual machine. actually, the command listener works based on a http server, command tools use http protocol to communicate with it.

#### command-executor

this entity only deploys on virutal machine endpoint, it receives command execution messages from vmbus-tunnel initiated by vmbusifyexec tool via command listener, then it will create process to run required program, redirect stdin and stdout if required, print on stdout could be returned to vmbusifyexec ontime. set timeout for executing process is supported and after process terminated, exit code will be returned to vmbusifyexec.

### socks-proxy approach

#### proxy-client

#### proxy-server

### virutal-subnet approach

#### ip-bearer

#### vir-ethio

#### nat
