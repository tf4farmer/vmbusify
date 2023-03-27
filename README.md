
# VMBusify(vmbusify) solution and specification

## 1 Lead

VMBusify(vmbusify), provide a way for communication and control between physical and virtual machines based on Microsoft VMBus architecture, this name combines 'VMBus' and 'simplify', and suggests that the software simplifies communication and control between the physical and virtual machines.

## 2 Requirements

we have two cases:

- after virtual machine created, usually, the SDN network is not ready immediately and the virtual machine is not accessible, in such a period before SDN network ready, we need an alternative way to set up the communications between the virtual machines and external network, then we can directly install software and perform initialization tasks on the virtual machine without SDN networking, shorten the virtual machine preparation time.

- when the SDN network encounters some failures, there would be no route from the virtual machines to access external network and also they are unreachable from external network, in such case, we need an alternative way to recover the communications between the virtual machines and external network, to repair the network fault, making the services on virtual machines available for winning a highly continuity, shorten the downtime.

in summary, an alternative way is very necessary for operations, communications and controls to virtual machines when SDN network is unavailable, based on Microsoft Hyper-V architecture, VMBus provide us such an alternative way and this project is based on this technology.

## 3 Approaches

for communication and control between physical and virtual machines based on VMBus technology, we provide the following three approaches.


### 3.1 command-execution

command line tool running on physical machine, similar with the PSExec tool, use it to connect to a target virtual machine and execute commands with stdin and stdout being redirected to the physical machine

typical usage:

		vmbusifyexec <VMID> [-it] <program> <args>

### 3.2 socks-proxy

a bidirectional socks proxy via vmbus as a underlay tunnel between the physical machine and virtual machines that it holds,	processes on the virtual machine can use this proxy to access the external network by the physical machine as the socks proxy server,	processes on the physical machine also can use this proxy to access the tcp services on the virtual machines

### 3.3 virtual-subnet

networking for the virtual machines and physical machine, a virtual subnet can be built among the physical machine and all virtual machines that it holds, with the physical machine as the NAT for this subnet connecting to external network, being different from normal network, this virtual subnet use vmbus as the physical layer instead of Ethernet and MAC

## 4 Architecture

in summary, below is the architecture overview illustration

```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


                                                                                                   +------------+
                                                                                                   |            |
                                                                                                   | httpserver <------------------------------------------------------+
                                                                                                   |            |                                                      |
                                                                                                   +-----^------+                                                      |
                                                                                                         |                                                             |
                                                                                                         |                                                             |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+
|                                                                          |        |                    |                                                             |        |
|                                                                          |        |                    |                                                             |        |
|      +----------+           +---------+      +----------+  +--------+    |        |   +------------+   |   +--------------------+  +-------------+                   |        |
|      | curl ... |           | ping... |      | wget ... |  |  sshd  |    |        |   | ssh client |   |   | vmbusifyexec ping..|  | vmbusifyctl |                   |        |
|      +----+-----+           +----^----+      +-----+----+  +---^----+    |        |   +-------+----+   |   +---+----------------+  +--+----------+                   |        |
|           |                      |                 |           |         |        |           |        |       |                      |                              |        |
|           |                      +-------+         |           |         |        |           |        |       |   +------------------+                              |        |
|           |                              |         |           |         |        |           |        |       |   |                                                 |        |
|           |                +--------+----+-----+---v----+------+-+       |        |       +---v----+---+----+--v---v---+--------+                                    |        |
|           |                |        |          |        |        |       |        |       |        |        |          |        |                                    |        |
|           |                | ip     | command  | proxy  | proxy  |       |        |       | proxy  | proxy  | command  | ip     |                                    |        |
|           |          +-----> bearer | executor | client | server |       |        |       | client | server | listener | bearer <----+                               |        |
|           |          |     |        |          |        |        |       |        |       |        |        |          |        |    |                               |        |
|           |          |     +--------+----------+--------+--------+       |        |       +--------+--------+----------+--------+    |                               |        |
|           |          |     |                                     |       |        |       |                                     |    |                               |        |
|           |          |     |           vmbus tunnel              |       |        |       |            vmbus tunnel             |    |                               |        |
|           |          |     |                                     |       |        |       |                                     |    |                               |        |
|           |          |     +-----------------^-------------------+       |        |       +------------------^------------------+    |                               |        |
|           |          |                       |                           |        |                          |                       |                               |        |
|           |          |                       |                           |        |                          |                       |                               |        |
|     +-----v------+   |               +-------v--------+                  |        |                  +-------+--------+              |                               |        |
|     |            |   |               |                |                  |        |                  |                |              |                               |        |
|     | tcp-socket |   |               |  vmbus-socket  |                  |        |                  |  vmbus-socket  |              |                               |        |
|     |            |   |               |                |                  |        |                  |                |              |                               |        |
|     +-----+------+   |               +-------^--------+                  |        |                  +-------^--------+              |                               |        |
|           |          |                       |                           |        |                          |                       |                               |        |
|           |          |                       |                           |        |                          |                       |                               |        |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+
|           |          |                       |                           |        |                          |                       |                               |        |
|           |          |                       |                           |        |                          |                       |                               |        |
|      +----v----+  +--+-----------+   +-------v--------+                  |        |                  +-------v--------+          +---v---------+  +----------+  +----+----+   |
|      |   tcp   |  |              |   |                |                  |        |                  |                |          |             |  |    tcp   |  |         |   |
|      +---------+  |   vir-ethio  |   |  vmbus-device  <---------------------------------------------->  vmbus-device  |          |  vir-ethio  |  +----------+  |   nat   |   |
|      |   ip    |  |              |   |                |                  |        |                  |                |          |             |  |    ip    |  |         |   |
|      +------^--+  +--^-----------+   +----------------+                  |        |                  +----------------+          +----------^--+  +--^----^--+  +---^-----+   |
|             |        |                                                   |        |                                                         |        |    |         |         |
|             |        |                                                   |        |                                                         |        |    |         |         |
|             +--------+                                                   |        |                                                         +--------+    +---------+         |
|                                                                          |        |                                                                                           |
|                                                        virtual machine   |        |  physical machine                                                                         |
|                                                                          |        |                                                                                           |
+--------------------------------------------------------------------------+        +-------------------------------------------------------------------------------------------+

```

### 4.1 vmbus-tunnel

this is the transport layer based on VMBus between physical and virtual machines, for each virtual machine, physical machine could establish one VMBus connection with it and provide TLV (Type-Length-Value) format package switch capability by time division multiplexing. we define the TLV format data transfer unit as vmbus package.

so, for the virtual machine endpoint, it will listen on VMBus and then accept the VMBus connection, only one connection would be kept, new connection would be accepted unless current connection being released.

for the physical machine, it will connect to a virtual machine on demand, also, user could make the physical machine connect to all virtual machines that it holds and keep the connections available for the full life cycle. typically, physical machine endpoint will keep a set indicating relationship from VMID(virtual machine identity) and IP address to the socket of VMBus connection.

virtual machine connection relationship triple-tuple:

		{<vmid, ip, socket>}

on physical machine endpoint, VMID is provide for upper layer to distinguish the target virutal machine to vmbus tunnel that it would like to send message to, and vmbus tunnel provide a service for upper layer to get VMID by virtual machine IP address. as the following illustration, physical machine manages multiple connections and only one connection for each virtual machine, for the vmbus tunnel, we only support sending and receiving messages between the virtual machine and physical machine, dispatching message from one virtual machine to another virtual machine via physical machine by vmbus tunnel is not support.


```
+-----------------+
|                 |                              <vmid1, ip1, sock1>
| virtual machine <--------------------------------+
|                 |                                |
+-----------------+                                |
                                                   |
                                                   |
+-----------------+                                |
|                 |           <vmid3, ip3, sock3>  |
| virtual machine <---------------+                |
|                 |               |                |
+-----------------+               |            +---+--------------+
                                  +------------+                  |
                                               | physical machine |
+-----------------+               +------------+                  |
|                 |               |            +---+--------------+
| virtual machine <---------------+                |
|                 |          <vmid4, ip4, sock4>   |
+-----------------+                                |
                                                   |
                                                   |
+-----------------+                                |
|                 |                                |
| virtual machine <--------------------------------+
|                 |                              <vmid2, ip2, sock2>
+-----------------+

```

multiplexing on each connection between virtual machine and physical machine requires the tunnel as a serialized transmission channel for all TLV format vmbus packages, this makes the data transmission mechanism simple and easy to understand, meanwhile, we provide both synchronized and asynchronized mode for sending data. for the upper layer of vmbus tunnel illustrated in the graph, such as proxy client/server, command listener/executor and so on, we define them as the application layer, for each entity in the application layer, we define the application layer transfer unit which means the payload of vmbus package as application message. as the illustration below, the hierarchy architecture contains two layers, the infrastructure is vmbus tunnel for transmitting vmbus-package and the upper layer is application layer for transmitting application-message.

```

+--------------------+         +--------------------+
|                    |         |                    |
|    application     +---------> application-message|
|                    |         |                    |
+--------------------+         +--------------------+
|                    |         |                    |
|    vmbus-tunnel    +--------->    vmbus-package   |
|                    |         |                    |
+--------------------+         +--------------------+

```

below is the illustration that indicating the elements of vmbus package, the 'sig' field is the signature number that represents the head of a package, 'type' indicates that whether the package is a normal request or an acknowledgement for a given request. 'id' is a counter for each package that sending from current endpoint while for an acknowledgement it will be same with its related request. 'message-id' and 'application-entity' decide the message handler for the application message to dispatch, 'length' indicates the bytes of payload which contains the application message.

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

vmbus tunnel provide a registry service for them to declare themselves by using their entity names and then, declare all message id they wish to receive and message handler for processing it. the primary key for message is <entity-name, message-id>, for both endpoint, the vmbus tunnel keeps a set of two-tuple, we call it application message route table.

application message route table two-tuple, the second element of the tuple is a set of triple-tuple that indicating all messages that the entity wish to receive and their message handlers.

		{<self-entity-name, {<message-source-entity-name, message-id, message-handler>}>}

message-id indicates what message they want and message-source-entity-name indicates from which entity they care, if and only if the given message from the given source entity would be targeted to the declared message handler. for each message, if there exists more than one handler, then each handler would be triggered serially in order of registration.


for sending and receiving data, on virtual machine endpoint, vmbus tunnel provide the following interfaces:

```
	int sync_send(int message_id, char const * data, int size, char ** out_buf_ptr, int * bytes_returned);
	int async_send(int message_id, char const * data, int size);
	typedef int (*message_handler)(int message_id, char const * data, int size, char ** out_buf_ptr, int * bytes_returned);
```

for sending and receiving data, on physical machine endpoint, vmbus tunnel provide the following interfaces:

```
	int sync_send(string const &vmid, int message_id, char const * data, int size, char ** out_buf_ptr, int * bytes_returned);
	int async_send(string const &vmid, int message_id, char const * data, int size);
	typedef int (*message_handler)(string const &vmid, int message_id, char const * data, int size, char ** out_buf_ptr, int * bytes_returned);
	string get_vmid_by_ip(char const * ip);
```

the intefaces quoted above are only a conceptual design, the sync_send and async_send are used for sending data to remote endpoint, message_handler is used for receiving message from remote endpoint. the difference bewteen physical and virtual machine is the vmid parameter is required by physical machine endpoint, because physical machine manages multiple vmbus connections and for sending message, it must provide the vmid of target virtual machine, for receiving message, it also will be informed the vmid indicating the message from which virutal machine sent. for the virtual machine endpoint, thanks to the restriction of only one connection with physical machine, target for sending and source for receiving is not needed.

### 4.2 command-execution approach

to sum up the command execution mechanism, the vmbusifyexec, command listener, vmbus tunnel and command executor get together to constitute an application for remote commands execution, such as the Windows internal tool 'PsExec'.

```
                            physical machine                                                                   virtual machine
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
        |                           |                           |                            |    | dispatch message    |                         |
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

#### 4.2.1 vmbusifyexec

this entity is a command line tool for executing command on virtual machines via the physical machine. it only deploys on physical machine endpoint. usage of this tool is similar to 'PsExec'.

#### 4.2.2 command-listener

this entity only deploys on physical machine endpoint, listen on tcp 127.0.0.1:6543, providing an operation interface for command line tools such as vmbusifyctl and vmbusifyexec. we defined two types of commands, the first class is built-in commands for controlling the vmbus tunnel, for example, connect to or disconnect from given target virtual machine, the second class is application specified commands, for example, execute ping command on given target virtual machine. actually, the command listener works based on a http server, command tools use http protocol to communicate with it.

#### 4.2.3 command-executor

this entity only deploys on virtual machine endpoint, it receives command execution messages from vmbus tunnel initiated by vmbusifyexec tool via command listener, then it will create process to run required program, redirect stdin and stdout if required, print on stdout could be returned to vmbusifyexec on time. set timeout for executing process is supported and after process terminated, exit code will be returned to vmbusifyexec.

### 4.3 socks-proxy approach

we provide a bidirectional socks proxy mechanism for user based on socksv5 protocol. for the virtual machine endpoint, user could use this approach to access external network by physical machine and for the physical machine, user could use this approach to access tcp service on the virtual machine. the following illustration is the typical routine for virtual machine to establish http connection with external http service by the socks proxy mechanism through vmbus tunnel.

for a common socks proxy server, it is a single process that listening on a tcp port for accepting socks proxy requests and generate a socket with user, then it will receive the real target ip and port by socks protocol via the socke with user, after that, it will connect to the real target ip and port and then generate a socket with target, then it has the relationship of the socket-with-user and socket-with-target, for data received from socket-with-user, then send to socket-with-target and for data received from socket-with-target, then send to socket-with-user. being different from such a common socks proxy server routine, in our vmbus context, the socks proxy server is divided into two parts, in other words two processes, proxy client that deploys on local endpoint and proxy server that deploys on remote endpoint. for the proxy client, it listens on a local tcp port and receives socks proxy request from user, for the proxy server, it is responsible for connecting to real target, so, the proxy client keeps the socket-with-user and the proxy server keeps the socket-with-target, and the bidirectional data is carried by the vmbus tunnel. the mapping relationship for the socket-with-user and socket-with-target is being accomplished by a proxy-id generated by proxy client for each socket-with-user.

generally, we defines the following proxy messages:

		PROXY-OPEN
			from proxy client to proxy server, indicating the target ip and port
		PROXY-OPEN-ACK:
			from proxy server to proxy client, acknowledgement of PROXY-OPEN, indicating whether the target connection established or not
		PROXY-DATA:
			bidirectional, for transmitting data, with proxy-id as the bridge between the socket-with-user and socket-with-target.
		PROXY-CLOSE:
			bidirectional, for disconnecting tcp connection and closing socket. when user shutdown and close socket, the socket-wit-user will be closed by proxy client, then this message will be send to proxy server on remote endpoint with the related proxy-id, to inform the proxy server shutdown and close the socket-with-target, meanwhile, when the socket-with-target disconnected, socket-wit-user also will be closed by proxy client after receiving PROXY-CLOSE message from proxy server.


```
                            virtual machine                                                        physical machine
+------------------------------------------------------------------------+          +-------------------------------------------+
+                                                                        +          +                                           +

+----------------+          +----------------+          +----------------+          +----------------+         +----------------+         +----------------+
|                |          |                |          |                |          |                |         |                |         |                |
|  wget b.com    |          |  proxy client  |          |  vmbus-tunnel  |          |  vmbus-tunnel  |         |  proxy server  |         |     b.com      |
|                |          |                |          |                |          |                |         |                |         |                |
+-------+--------+          +-------+--------+          +-------+--------+          +--------+-------+         +--------+-------+         +-------+--------+
        |                           |                           |                            |                          |                         |
        |        tcp connect        |                           |                            |                          |                         |
        +--------------------------->                           |                            |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           +-----+                     |                            |                          |                         |
        |                           |     |sock=accept()        |                            |                          |                         |
        |                           |     |generate proxyid     |                            |                          |                         |
        |                           <-----+                     |                            |                          |                         |
        |       connect ack         |                           |                            |                          |                         |
        <---------------------------+                           |                            |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          |                         |
        |     scoskv5 handshake     |                           |                            |                          |                         |
        +--------------------------->                           |                            |                          |                         |
        |        ip,port            |                           |                            |                          |                         |
        |                           |            call           |                            |                          |                         |
        |                           +--------------------------->                            |                          |                         |
        |                           |     proxy-open message    |                            |                          |                         |
        |                           |      proxyid,ip,port      |        vmbus socket        |                          |                         |
        |                           |                           +---------------------------->                          |                         |
        |                           |                           |       vmbus package        |                          |                         |
        |                           |                           |                            +----+                     |                         |
        |                           |                           |                            |    | dispatch message    |                         |
        |                           |                           |                            |    |                     |                         |
        |                           |                           |                            <----+                     |                         |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |        callback          |                         |
        |                           |                           |                            +-------------------------->                         |
        |                           |                           |                            |    proxy-open message    |                         |
        |                           |                           |                            |                          +----+                    |
        |                           |                           |                            |                          |    | get vmid,proxyid   |
        |                           |                           |                            |                          |    | and ip,port        |
        |                           |                           |                            |                          <----+                    |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          |      tcp connect        |
        |                           |                           |                            |                          +------------------------->
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          |      connect ack        |
        |                           |                           |                            |                          <-------------------------+
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |                          +----+ get sock to b.com  |
        |                           |                           |                            |                          |    | map vmid,proxyid   |
        |                           |                           |                            |                          |    | with sock          |
        |                           |                           |                            |                          <----+                    |
        |                           |                           |                            |                          |                         |
        |                           |                           |                            |          call            |                         |
        |                           |                           |                            <--------------------------+                         |
        |                           |                           |                            |  proxy-open-ack message  |                         |
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
        |                           |  proxy-open-ack message   |                            |                          |                         |
        |     socksv5 handshake     |                           |                            |                          |                         |
        <---------------------------+                           |                            |                          |                         |
        |          ack              |                           |                            |                          |                         |
        +                           +                           +                            +                          +                         +


```

#### 4.3.1 proxy-client

the proxy-client deploys on both the virtual machine and physical machine endpoint, listening on 127.0.0.1:6542, providing socksv5 protocol for user processes. after accepting a connection, the socket-with-user generated, the proxy client generates a unique proxy-id binding to the socket-with-user, so, for the proxy client on the virtual machine endpoint, it holds a set of two-tuple, that is:

		{<socket-with-user, proxy-id>}

for data received from socket-with-user, proxy client will send it to vmbus tunnel with the proxy-id, for data received from vmbus tunnel, proxy client will unpack it and get the proxy-id, then get the socket-with-user by proxy-id and send data to user by the socket-with-user.

for proxy client on the physical machine endpoint, there's a little difference. after socket-with-user generated, proxy client will get the target ip by socks protocol, then, proxy client should call the get_vmid_by_ip interface provided by vmbus tunnel which mentioned in above section, to get the target virtual machine vmid. so, for the proxy client on the physical machine endpoint, it holds a set of triple-tuple, that is:

		{<socket-with-user, proxy-id, vmid>}

for data received from socket-with-user, proxy client will send it to vmbus tunnel with the proxy-id to the target virtual machine distinguished by vmid, for data received from vmbus tunnel, same with proxy client on virtual machine endpoint, it only needs the proxy-id to get the socket-with-user and then send data to user.

in addition, we also provide a socks hook dynamic link library which can replace the original call of socket interfaces of a program and redirect the connection to socks proxy client, that making the socks proxy totally transparent for the program and user. on Windows we use detour mechanism and on Linux we use RTLD_NEXT trick to achieve this goal. user could use this library for those programs that do not support socksv5 protocol. another advantage is that by hooking socket interfaces via this library, user's process do not care whether the network is available or not, when the network is down and the vmbus tunnel is ready, this library would redirect tcp connections to be carried by socks protocol by using vmbus tunnel, otherwise, this library would directly dispatch user's call to original socket interface. this makes the communication can automatically and transparently switch from tcp/ip network to vmbus tunnel.

```
                    PRELOAD and RTLD_NEXT
                              +
+-----------------+           |           +-----------------+
|                 |           |           |                 |
|   wget b.com    |           |           |   wget b.com    |
|                 |           |           |                 |
+--------+--------+           |           +--------+--------+
         |                    |                    |
         |                +---v--->                |
         |                                         |
+--------v--------+       +------->       +--------v--------+             +-----------------+
|                 |                       |                 |             |                 |
|     libc.so     |                       |     proxy.so    +------------->   proxy client  |
| int connect(..) |                       | int connect(..) |             | listen on 6542  |
|                 |                       |                 |             |                 |
+-----------------+                       +-----------------+             +-----------------+

```


#### 4.3.2 proxy-server

the proxy server deploys on both the virtual machine and physical machine endpoint, after received the PROXY-OPEN message, it will get the proxy-id, target ip and port and the vmid where this message come from, then it will try to connect to the target, and then response to proxy client by PROXY-OPEN-ACK message to inform if the connection being established or not.

for the proxy-server on virtual machine endpoint, it holds a set of two-tuple, that is:

		{<socket-with-target, proxy-id>}
		
for the proxy server on physical machine endpoint, it holds a set of triple-tuple, that is:

		{<socket-with-target, proxy-id, vmid>}

### 4.4 virutal-subnet approach

the main purpose is to make the physical machine and virtual machines combined as a virtual subnet with the physical machine as the default gateway for each virtual machine and the router of this virtual subnet.

#### 4.4.1 ip-bearer

this application entity is used for transmitting IP packages by using vmbus tunnel, for data production, it opens the vir-ethio device and holds its file descriptor, then read IP packages from vir-ethio device and send to remote endpoint, for data consumption, it receives IP packages from vmbus tunnel, then writes to vir-ethio device. 

for the proxy server endpoint, besides the responsibility for IP packages dispatching, it has further more tasks, that is, management and assignment virtual subnet IP addresses for all virtual machines. on the initialization stage after vmbus tunnel established with a given virtual machine, ip-bearer application on physical machine endpoint will send a IP assignment message to the virtual machine, the virtual machine then will enable the vir-ethio and set the given IP address to it, then update the ip table to make all required IP packages route to the vir-ethio device.

for physical endpoint, it holds a set of two-tuple, that is:

		{<virtual-subnet-ip, vmid>}

#### 4.4.2 vir-ethio

typically, this is a tun device, with the IP address assigned by ip-bearer that given by physical meachine endpoint, created and managed by ip-bearer entity.

#### 4.4.3 nat

typically, this is also a tun device. by using a free software with proper configurations, we can make the bidirectional access available, not only from virtual machine to external network but also from external network to virtual machine. 

