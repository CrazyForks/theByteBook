# 3.5.4 虚拟交换机 Linux bridge

在物理网络中，通常使用交换机连接多台主机，组成小型局域网。Linux 网络虚拟化技术中，也提供了物理交换机的虚拟实现 —— Linux bridge（Linux 网桥，也称虚拟交换机）。

Linux bridge 作为虚拟交换机，功能与物理交换机类似。通过 brctl 命令将多个网络接口（如物理网卡 eth0、虚拟接口 veth、tap 等）桥接后，它们的通信方式与物理交换机的转发行为一致。当数据帧进入 Linux bridge 时，系统根据数据帧的类型和目的地 MAC 地址执行以下处理：
- 对于广播帧，转发到所有桥接到该 Linux bridge 的设备。
- 对于单播帧，查找 FDB（Forwarding Database，地址转发表）中 MAC 地址与设备网络接口的映射记录：
	- 若未找到记录，执行“洪泛”（Flooding），然后根据“泛红”响应将设备的网络接口与 MAC 地址记录到 FDB 表中；
	- 若找到记录，直接将数据帧转发到对应设备的网络接口。

举一个具体的例子，使用 Linux bridge 将两个命名空间接入到同一个二层网络。该例子的网络拓扑结构如图 3-15 所示。

:::center
  ![](../assets/linux-bridge.svg)<br/>
 图 3-15 veth 网卡与 Linux Bridge
:::

创建 Linux bridge 与其他虚拟网络设备类似，只需将 type 参数指定为 bridge 即可。

```bash
$ ip link add name br0 type bridge
$ ip link set br0 up
```

创建的 Linux bridge 一端连接到主机协议栈，而其他端口尚未连接任何设备。为了实现其功能，需要将其他设备连接到该 bridge。接下来，我们将创建网络命名空间和 veth 设备，并将 veth 的一端连接到网络命名空间，另一端连接到刚创建的 br0 桥接设备。

```bash
# 创建网络命名空间
$ ip netns add ns1
$ ip netns add ns2

# 创建 veth 网线
$ ip link add veth0 type veth peer name veth1
$ ip link add veth2 type veth peer name veth3

# 将 veth 网线的一端连接到网络命名空间内
$ ip link set veth0 netns ns1
$ ip link set veth2 netns ns2

# 将 veth 另一端连接到 br0
$ ip link set dev veth1 master br0
$ ip link set dev veth3 master br0
```

激活网络命名空间内的虚拟网卡，并为它们设置 IP 地址。这些 IP 地址位于同一个子网 172.16.0.0/24 中。

```bash
# 配置命名空间1
$ ip netns exec ns1 ip link set veth1 up
$ ip netns exec ns1 ip addr add 172.16.0.1/24 dev veth1
# 配置命名空间2
$ ip netns exec ns2 ip link set veth2 up
$ ip netns exec ns2 ip addr add 172.16.0.2/24 dev veth2
```

接下来，检查网络命名空间之间是否可达。

```bash
ip netns exec ns1 ping 172.16.0.2
PING 172.16.0.2 (172.16.0.2) 56(84) bytes of data.
64 bytes from 172.16.0.1: icmp_seq=1 ttl=64 time=0.153 ms
64 bytes from 172.16.0.1: icmp_seq=2 ttl=64 time=0.148 ms
64 bytes from 172.16.0.1: icmp_seq=3 ttl=64 time=0.116 ms
```

通过上述实验，我们验证了使用 Linux bridge 可以将多个命名空间连接到同一个二层网络中。

最后需要补充的是，Linux bridge 本质上是 Linux 系统中的虚拟网络设备，具备网卡特性，能够配置 MAC 和 IP 地址。从主机的角度来看，配置了 IP 地址的 Linux bridge 设备就相当于一张网卡，能够参与数据包的 IP 路由。因此，当网络命名空间的默认网关设置为 Linux bridge 的 IP 地址时，原本隔离的网络命名空间便能够与主机进行通信。

实现容器与主机之间的互通，是容器间通信的关键环节。这方面的内容，笔者将在第七章的 7.6 节中详细阐述。


