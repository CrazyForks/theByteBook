# 4.3 四层负载均衡

前面提到的“四层负载均衡”（Layer4 Balancer，简称 L4）其实是多种均衡器工作模式的统称，“四层”的意思是这些工作的模式共同特点是维持同一个 TCP 连接，而不是说它只工作在 OSI 模型的四层。

事实上，四层负载均衡器的多种模式主要工作在第二层（数据链路层，改写 MAC 地址）和第三层（网络层，改写 IP 地址），而第四层（传输层）单纯只改写一些 UDP、TCP 等协议的内容端口，做一些 NAT 之类的功能，实际上无法实现字面上的“负载均衡”。

因为 OSI 模型的下三层是媒体层（Media Layer），上四层是主机层（Host Layer），既然流量已经到了目标主机了，也谈不上什么流量转发，最多只能做代理，但出于习惯，现在所有的资料都把它们称为四层负载均衡。

我也沿用这种惯例，不论是 IP 层处理还是链路层处理，就统称四层负载均衡吧。
