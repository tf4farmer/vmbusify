
# VMBusify(vmbusify) solution and specification

## Lead

VMBusify(vmbusify), provide a way for communication and control between physical and virtual machines based on Microsoft VMBus architecture, this name combines 'VMBus' and 'simplify', and suggests that the software simplifies communication and control between the physical and virtual machines.

## Requirements

we have two cases:

1. after virtual machine created, usually, the SDN network is not ready immediately and the virtual machine is not accessible, in such a period before SDN network ready, we need an alternative way to set up the communications between the virtual machines and external network, then we can directly install softwares and perform initialization tasks on the virutal machine without SDN networking, shorten the virtual machine preparation time.

2. when the SDN network encouters some failures, there would be no route from the virtual machines to access external network and also they are unreachable from external network, in such case, we need an alternative way to recover the communications between the virtual machines and external network, to repair the network fault, making the services on virtual machines available for winning a highly continuity, shorten the downtime.

in summary, an alternative way is very necessary for operations, commnunications and controls to virtual machines when SDN network is unavalable, based on Microsoft Hyper-V architecture, VMBus provide us such an alternative way and this project is based on this technology.

## Approach

for commnucation and control between physical and virtual machines based on VMBus technology, we provide the following three approaches.


### 1. vmbusify-exec

command line tool running on physical machine, similar with the PSExec tool, use it to connect to a target virtual machine and execute commands with stdin and stdout being redirected to the physical machine

typical usage:

		vmbusexec <VMID> [-it] <program> <args>

### 2. vmbusify-proxy

a bidirectional socks proxy via vmbus as a underlay tunnel between the physical machine and virtual machines that it holds,	processes on the virtual machine can use this proxy to access the external network by the physical machine as the socks proxy server,	processes on the physical machine also can use this proxy to access the tcp services on the virtual machines

### 3. vmbusify-vnet

networkng for the virtual machines and physical machine, a virutal subnet can be built among the physical machine and all virutal machines that it holds, with the physical machine as the NAT for this subnet connecting to external network, being different from normal network, this virutal subnet use vmbus as the physical layer instead of Ethernet and MAC

## Architecture

in summary, below is the architecture overview illustration

```
+-----------------------------------------------------------------------+        +----------------------------------------------------------------------------+
|                                                                       |        |                                                                            |
|                                                                       |        |                                                                            |
|                                                                       |        |                                                                            |
|                                                                       |        |                                                                            |
|                          +---------+      +----------+  +--------+    |        |   +------------+   +-------------+     +--------------------+              |
|                          | ping... |      | browser  |  |  sshd  |    |        |   | ssh client |   | httpservice |     | vmbusexec ping ... |              |
|                          +----+----+      +-----+----+  +---^----+    |        |   +-----+------+   +-----^-------+     +-----+--------------+              |
|                               |                 |           |         |        |         |                |                   |                             |
|                               +-------+         |           |         |        |         +------+         |         +---------+                             |
|                                       |         |           |         |        |                |         |         |                                       |
|                         +--------+----+-----+---v----+------+-+       |        |            +---v----+----+---+-----v----+--------+                         |
|                   +-----> ip     | command  | proxy  | proxy  |       |        |            | proxy  | proxy  | command  | ip     <----+                    |
|                   |     | bearer | executor | client | server |       |        |            | client | server | listener | bearer |    |                    |
|                   |     +--------+----------+--------+--------+       |        |            +--------+--------+----------+--------+    |                    |
|                   |     |                                     |       |        |            |                                     |    |                    |
|                   |     |           vmbus tunnel              <----------------------------->              vmbus tunnel           |    |                    |
|                   |     |                                     |       |        |            |                                     |    |                    |
|                   |     +-------------------------------------+       |        |            +-------------------------------------+    |                    |
|                   |                                                   |        |                                                       |                    |
+-----------------------------------------------------------------------+        +----------------------------------------------------------------------------+
|     +---------+   |                                                   |        |                                                       |     +---------+    |
|     |   tcp   |   |                                                   |        |                                                       |     |   tcp   |    |
|     +---------+   |                                                   |        |                                                       |     +---------+    |
|     |   ip    |   |                                                   |        |                                                       |     |   ip    |    |
|     +------^--+   |                                                   |        |                                                       |     +-^-------+    |
|            |      |                                                   |        |                                                       |       |            |
|            |      |                                                   |        |                                                       |       |            |
|           +v------v---+                                               |        |                                                    +--v-------v+           |
|           | vir|ethio |                                               |        |                                                    | vir-ethio |           |
|           +-----------+                 virtual machine               |        |     physical machine                               +-----------+           |
|                                                                       |        |                                                                            |
+-----------------------------------------------------------------------+        +----------------------------------------------------------------------------+

```
