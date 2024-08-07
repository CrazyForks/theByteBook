# 2.4 HTTP 请求优化

解决域名解析环节中的问题之后，我们继续看 HTTP 请求过程中有哪些方面的优化。

根据笔者的实践经验来看，关注以下几个方面的优化，能起到事半功倍的效果。

- 包体积优化：传输数据的包体大小与传输耗时成正相关，**减小包体**是优化最有效的手段之一。
- 传输层优化：**升级拥塞控制算法**（例如默认的 Cubic 升级为 BBR 算法）提高网络吞出率。
- 网络层优化：使用商业化的网络加速服务，对**数据包进行路由优化**，实现动态服务加速。
- SSL 层优化：**升级 TLS 算法**以及 HTTPS 证书，降低 SSL 层的计算消耗。
- 使用更现代的 HTTP 协议：升级至 HTTP/2，进一步可以使用 QUIC。

HTTP 服务优化，就从降低包体积开始。