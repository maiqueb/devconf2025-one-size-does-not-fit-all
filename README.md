# devconf2025-one-size-does-not-fit-all
Repo with instructions and manifests for the DevConf 2025 talk [Kubernetes Networking: One size doesn't fit all](https://pretalx.devconf.info/devconf-cz-2025/talk/LVKTGW/).

## Requirements
- [kind](https://kind.sigs.k8s.io/)
- [helm](https://helm.sh/)
- container runtime (docker / podman)

## Cluster creation instructions
1. Create a kind cluster (no CNI installed please).
```
kind create cluster --config manifests/empty-kind.yaml
kind export kubeconfig --name ovn-k-helm
```
2. Label the kind nodes
```
for node in $(kind get nodes --name "ovn-k-helm"); do
  kubectl label node "${node}" k8s.ovn.org/zone-name=${node} --overwrite
done
```
3. Clone the ovn-kubernetes repo
```
git clone https://github.com/ovn-kubernetes/ovn-kubernetes.git
```
4. Install OVN-Kubernetes in your kind cluster
```
cd ovn-kubernetes/helm/ovn-kubernetes
helm install . --generate-name -f ./values-single-node-zone.yaml \
    --set k8sAPIServer="https://$(kubectl get pods -n kube-system -l component=kube-apiserver -o jsonpath='{.items[0].status.hostIP}'):6443" \
    --set ovnkube-identity.replicas=$(kubectl get node -l node-role.kubernetes.io/control-plane --no-headers | wc -l) \
    --set global.image.repository=ghcr.io/ovn-kubernetes/ovn-kubernetes/ovn-kube-ubuntu \
    --set global.image.tag=master \
    --set global.enableMultiNetwork="true" \
    --set global.enableNetworkSegmentation="true"
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/refs/heads/master/deployments/multus-daemonset.yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multi-networkpolicy/refs/tags/v1.0.1/scheme.yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/ipamclaims/v0.4.0-alpha/artifacts/k8s.cni.cncf.io_ipamclaims.yaml

# lets wait for the availability of the ovnkube-node daemonset on all nodes ...
kubectl rollout status daemonset -novn-kubernetes ovnkube-node --timeout 300s
```
5. Install KubeVirt in your kind cluster
```
# install kubevirt
export RELEASE=$(curl https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-operator.yaml"
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${RELEASE}/kubevirt-cr.yaml"
kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout=300s 

# install ipam extensions
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait -ncert-manager deployment cert-manager-webhook \
    --for condition=Available \
    --timeout=60s

kubectl apply -f https://raw.githubusercontent.com/kubevirt/ipam-extensions/main/dist/install.yaml
kubectl wait -nkubevirt-ipam-controller-system deployment kubevirt-ipam-controller-manager \
    --for condition=Available \
    --timeout=60s

# configure additional network binding for primary UDN
kubectl patch -n kubevirt kubevirt kubevirt \
    --type=json \
    --patch '[
        {"op":"add","path":"/spec/configuration/network","value":{}},
        {"op":"add","path":"/spec/configuration/network/binding","value":{"l2bridge":{"domainAttachmentType":"managedTap","migration":{}}}}
    ]'
```

## Pod workshop

This part of the demo will get the participants familiar with the concept of UserDefinedNetwork (UDN). We will explore the following in detail:

* What is a UDN and a CUDN
* Creating UDNs and CUDNs to do network segmentation of a Kubernetes cluster
* Creating pods and attaching them to these UDNs
* Testing native network isolation across different UDNs
* Creating NodePprt type services in these UDNs
* Testing connectivity of clusterIP, nodePorts in these UDNs
* Creating NetworkPolicy in these UDNs and testing connectivity
* Ensuring Ingress and Egress works as expected for these UDN Pods
* Concept of multi-homing for a pod in Kubernetes using UDNs
* Creating pods in diffeernt UDNs with overlapping podIPs
* All the commands required to be executed on your KIND cluster are provided [here](hhttps://github.com/maiqueb/devconf2025-one-size-does-not-fit-all/blob/main/manifests/udns-with-pods/commands-cheatsheet-for-participants.md).

Workshop instruction manual that will be followed can be found [here](https://github.com/maiqueb/devconf2025-one-size-does-not-fit-all/blob/main/manifests/udns-with-pods/workshop-instructions-script.sh).


## VM workshop
This part of the demo will focus on the virtualisation use cases. We will create
a cluster UDN, spanning across multiple namespaces, start a VM in each namespace,
and show east/west connectivity between them, as well as connectivity to the
outside world.

Afterwards, we will showcase VM live-migration between nodes, explain how we
ensure the VM IPAM configuration does not change during the migration, and
which shenanigans (yes, there are shenanigans ...) we have to perform to
preserve the established TCP connections during migration, for both IPv4 and
IPv6 IP families.

1. Apply the UDN and the workload manifests
```sh
kubectl apply -f manifests/virt/01-udn.yaml
kubectl apply -f manifests/virt/02-workloads.yaml
```

2. Wait for the VMs to be running
```sh
kubectl wait vmi -nred-namespace red --for=jsonpath='{.status.phase}'=Running
kubectl wait vmi -nblue-namespace blue --for=jsonpath='{.status.phase}'=Running
```

3. Log into the VMs and ensure egress works as expected
Once the VMs are `Running`, we can now log into them via the console, using the `virtctl` CLI.

```sh
[root@fc40 ~]# virtctl console -nred-namespace red
Successfully connected to red console. The escape sequence is ^]

red login: fedora
Password:
[fedora@red ~]$ ip r
default via 192.168.0.1 dev eth0 proto dhcp metric 100
192.168.0.0/16 dev eth0 proto kernel scope link src 192.168.0.3 metric 100
[fedora@red ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:58:c0:a8:00:03 brd ff:ff:ff:ff:ff:ff
    altname enp1s0
    inet 192.168.0.3/16 brd 192.168.255.255 scope global dynamic noprefixroute eth0
       valid_lft 3116sec preferred_lft 3116sec
    inet6 fe80::858:c0ff:fea8:3/64 scope link
       valid_lft forever preferred_lft forever
[fedora@red ~]$ ping -c 2 www.google.com
PING www.google.com (142.250.178.164) 56(84) bytes of data.
64 bytes from 142.250.178.164 (142.250.178.164): icmp_seq=1 ttl=56 time=32.3 ms
64 bytes from 142.250.178.164 (142.250.178.164): icmp_seq=2 ttl=56 time=31.3 ms

--- www.google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 31.340/31.795/32.250/0.455 ms
```

4. East/west traffic over the UDN
Let's also ensure the east/west traffic works as expected in the UDN. For that, we first need to figure out
the IP address of one VM, and ping from the other.
```sh
[root@fc40 ~]# kubectl get vmi -n blue-namespace blue -ojsonpath="{@.status.interfaces}" | jq
[
  {
    "infoSource": "domain, guest-agent",
    "interfaceName": "eth0",
    "ipAddress": "192.168.0.4",
    "ipAddresses": [
      "192.168.0.4"
    ],
    "linkState": "up",
    "mac": "0a:58:c0:a8:00:04",
    "name": "happy",
    "podInterfaceName": "ovn-udn1",
    "queueCount": 1
  }
]

# now from the other VM:
[root@fc40 ~]# virtctl console -nred-namespace red
Successfully connected to red console. The escape sequence is ^]

[fedora@red ~]$
[fedora@red ~]$ ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4) 56(84) bytes of data.
64 bytes from 192.168.0.4: icmp_seq=1 ttl=64 time=6.56 ms
64 bytes from 192.168.0.4: icmp_seq=2 ttl=64 time=3.33 ms

--- 192.168.0.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 3.327/4.943/6.560/1.616 ms
```

5. VM live-migration
In this scenario, we will establish a TCP connection between both red and blue
VMs using `iperf3`, then migrate the blue VM to another Kubernetes node. We
will see that the established TCP connection will survive the migration, while
a few packets will be lost.

Keep in mind we've seen in the previous step the `blue` VM IP address is
`192.168.0.4`.

Let's log into the blue VM and establish an `iperf3` server; as with the `red`
VM, the user/password is `fedora`/`fedora`.
```sh
[root@fc40 ~]# virtctl console -nblue-namespace blue
Successfully connected to blue console. The escape sequence is ^]

blue login: fedora
Password:
[fedora@blue ~]$ iperf3 -s -1 -p9000
-----------------------------------------------------------
Server listening on 9000 (test #1)
-----------------------------------------------------------
```

Let's now access the `red` VM, and connect to the `blue` VM:
```sh
[root@fc40 ~]# virtctl console -nred-namespace red
Successfully connected to red console. The escape sequence is ^]

--- 192.168.0.4 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 3.327/4.943/6.560/1.616 ms
[fedora@red ~]$ iperf3 -c 192.168.0.4 -p 9000 -t 3600
Connecting to host 192.168.0.4, port 9000
[  5] local 192.168.0.3 port 47654 connected to 192.168.0.4 port 9000
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   564 MBytes  4.73 Gbits/sec    0   3.08 MBytes
[  5]   1.00-2.00   sec   452 MBytes  3.78 Gbits/sec    1   3.08 MBytes
[  5]   2.00-3.00   sec   616 MBytes  5.18 Gbits/sec    0   3.08 MBytes
[  5]   3.00-4.00   sec   426 MBytes  3.58 Gbits/sec    1   3.08 MBytes
[  5]   4.00-5.02   sec   588 MBytes  4.86 Gbits/sec    1   3.08 MBytes
[  5]   5.02-6.00   sec   585 MBytes  4.98 Gbits/sec    1   3.08 MBytes
...
```

Let's now issue the migrate command using the `virtctl` CLI:
```sh
[root@fc40 ~]# virtctl migrate -nblue-namespace blue
VM blue was scheduled to migrate
[root@fc40 ~]# kubectl get pods -nblue-namespace -w
NAME                       READY   STATUS            RESTARTS   AGE
virt-launcher-blue-crvf7   2/2     Running           0          29h
virt-launcher-blue-pzsrk   0/2     PodInitializing   0          8s
virt-launcher-blue-pzsrk   2/2     Running           0          12s
virt-launcher-blue-crvf7   1/2     NotReady          0          29h
virt-launcher-blue-pzsrk   2/2     Running           0          40s
virt-launcher-blue-pzsrk   2/2     Running           0          40s
virt-launcher-blue-pzsrk   2/2     Running           0          40s
virt-launcher-blue-pzsrk   2/2     Running           0          41s
virt-launcher-blue-crvf7   0/2     Completed         0          29h
virt-launcher-blue-crvf7   0/2     Completed         0          29h
```

You'll be ejected from the `blue` VM console, and you'll see something similar
to this in the `red` VM:
```sh
...
[  5]  70.00-71.03  sec   231 MBytes  1.89 Gbits/sec    0   1.90 MBytes
[  5]  71.03-72.01  sec  81.2 MBytes   698 Mbits/sec    0   2.02 MBytes
[  5]  72.01-73.02  sec   112 MBytes   934 Mbits/sec    0   2.08 MBytes
[  5]  73.02-74.00  sec   100 MBytes   852 Mbits/sec    2   1.32 KBytes
[  5]  74.00-75.02  sec  0.00 Bytes  0.00 bits/sec      1   1.32 KBytes
[  5]  75.02-76.00  sec  25.0 MBytes   213 Mbits/sec   24   1.32 KBytes
[  5]  76.00-77.00  sec   341 MBytes  2.87 Gbits/sec    3   3.01 MBytes
...
```
This concludes our live-migration demo.

