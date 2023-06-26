# tc_redir

## 获取代码

git clone后，执行

```bash
git submodule update --init --recursive
```

## netns setup

用root权限执行`netns.sh`，随后用`ip l`查看两对veth中在host namespace的ifindex，并修改

* tc_redir_libbpf/examples/c/tc.bpf.c
* tc_redir_libbpf/examples/c/tc.c

两个文件中`VETH1_IFINDEX`和`VETH2_IFINDEX`的定义。

## 编译和加载bpf程序

进入` tc_redir_libbpf/examples/c`目录，使用CMake编译

```bash
mkdir build
cd build
cmake ..
make
```

随后用root权限运行编译产物`./tc`加载bpf程序。

## benchmark

使用`sudo ip netns exec`在指定的netns中运行程序。首先验证两个netns间是否可以ping通：

```bash
sudo ip netns exec ns1 ping 10.10.0.20
sudo ip netns exec ns2 ping 10.10.0.10
```

然后在两个netns中运行fortio benchmark。

## baseline

执行`netns_norm.sh`，创建的两个netns分别为`ns3`和`ns4`，IP分别为`10.10.0.30`和`10.10.0.40`

## TODOs

- [ ] benchmark results
- [ ] 使用BPF global variable或Maps存储veth间的“路由”信息，即veth的IP（实际上是vpeer的IP）和ifindex间的对应关系，而非使用宏硬编码
- [ ] 在bpf程序中，根据skb的目标IP判断是否可以使用`bpf_redirect_peer`转发（即同一host不同netns间通信），若不能（即访问外部网络通信），使用`bpf_redirect_neigh`转发到host的eth0或相应网卡，并且注意需要实现iptables的MASQUERADE功能，以让包可以正确被外部设备处理（有可能还是需要加回bridge设备）
- [ ] 在bpf中正确实现ARP转发，或满足neighbouring subsystem。目前在`netns.sh`中使用手工添加的neighbor绕过了neighbor subsystem（第59-60行），若去掉这两行将会导致ARP不通，进而无法使用IP或TCP访问其他netns