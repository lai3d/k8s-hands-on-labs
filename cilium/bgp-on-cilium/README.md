
```bash
root@server:~# kind create cluster --config cluster.yaml
Creating cluster "clab-bgp-cplane-demo" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼 
 ✓ Preparing nodes 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-clab-bgp-cplane-demo"
You can now use your cluster with:

kubectl cluster-info --context kind-clab-bgp-cplane-demo

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
root@server:~# cat cluster.yaml
kind: Cluster
name: clab-bgp-cplane-demo
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: "10.1.0.0/16"
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: "10.0.1.2"
        node-labels: "rack=rack0"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: "10.0.2.2"
        node-labels: "rack=rack0"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: "10.0.3.2"
        node-labels: "rack=rack1"
- role: worker
  kubeadmConfigPatches:
  - |
    kind: JoinConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-ip: "10.0.4.2"
        node-labels: "rack=rack1"
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:5000"]
    endpoint = ["http://kind-registry:5000"]
root@server:~# kubectl get nodes
NAME                                 STATUS     ROLES           AGE     VERSION
clab-bgp-cplane-demo-control-plane   NotReady   control-plane   3m35s   v1.25.3
clab-bgp-cplane-demo-worker          NotReady   <none>          3m10s   v1.25.3
clab-bgp-cplane-demo-worker2         NotReady   <none>          3m10s   v1.25.3
clab-bgp-cplane-demo-worker3         NotReady   <none>          3m10s   v1.25.3
root@server:~# kubectl get nodes -o wide
NAME                                 STATUS     ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION    CONTAINER-RUNTIME
clab-bgp-cplane-demo-control-plane   NotReady   control-plane   4m29s   v1.25.3   <none>        <none>        Ubuntu 22.04.1 LTS   5.15.0-1022-gcp   containerd://1.6.9
clab-bgp-cplane-demo-worker          NotReady   <none>          4m4s    v1.25.3   <none>        <none>        Ubuntu 22.04.1 LTS   5.15.0-1022-gcp   containerd://1.6.9
clab-bgp-cplane-demo-worker2         NotReady   <none>          4m4s    v1.25.3   <none>        <none>        Ubuntu 22.04.1 LTS   5.15.0-1022-gcp   containerd://1.6.9
clab-bgp-cplane-demo-worker3         NotReady   <none>          4m4s    v1.25.3   <none>        <none>        Ubuntu 22.04.1 LTS   5.15.0-1022-gcp   containerd://1.6.9
root@server:~# 
```

💻 Networking Fabric
To showcase the Cilium BGP feature, we need a BGP-capable device to peer with.

For this purpose, we will be leveraging Containerlab and FRR (Free Range Routing). 
These great tools provide the ability to simulate networking environment in containers.


🧪 Containerlab
Containerlab is a platform that enables users to deploy virtual networking topologies, based on containers and virtual machines. 
One of the virtual routing appliances that can be deployed via Containerlab is FRR - a feature-rich open-source networking platform.

By the end of the lab, you will have established BGP peering with the FRR virtual devices.

```bash
root@server:~# bash -c "$(curl -sL https://get.containerlab.dev)" -- -v 0.31.1
Downloading https://github.com/srl-labs/containerlab/releases/download/v0.31.1/containerlab_0.31.1_linux_amd64.deb
Preparing to install containerlab 0.31.1 from package
Selecting previously unselected package containerlab.
(Reading database ... 64446 files and directories currently installed.)
Preparing to unpack .../containerlab_0.31.1_linux_amd64.deb ...
Unpacking containerlab (0.31.1) ...
Setting up containerlab (0.31.1) ...

                           _                   _       _     
                 _        (_)                 | |     | |    
 ____ ___  ____ | |_  ____ _ ____   ____  ____| | ____| | _  
/ ___) _ \|  _ \|  _)/ _  | |  _ \ / _  )/ ___) |/ _  | || \ 
( (__| |_|| | | | |_( ( | | | | | ( (/ /| |   | ( ( | | |_) )
\____)___/|_| |_|\___)_||_|_|_| |_|\____)_|   |_|\_||_|____/ 

    version: 0.31.1
     commit: 4fbe732d
       date: 2022-08-15T13:18:12Z
     source: https://github.com/srl-labs/containerlab
 rel. notes: https://containerlab.dev/rn/0.31/#0311
root@server:~# cat topo.yaml
name: bgp-cplane-demo
topology:
  kinds:
    linux:
      cmd: bash
  nodes:
    router0:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
      # NAT everything in here to go outside of the lab
      - iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
      # Loopback IP (IP address of the router itself)
      - ip addr add 10.0.0.0/32 dev lo
      # Terminate rest of the 10.0.0.0/8 in here
      - ip route add blackhole 10.0.0.0/8 
      # Boiler plate to make FRR work
      - touch /etc/frr/vtysh.conf
      - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
      - /usr/lib/frr/frrinit.sh start
      # FRR configuration
      - >-
         vtysh -c 'conf t'
         -c 'frr defaults datacenter'
         -c 'router bgp 65000'
         -c '  bgp router-id 10.0.0.0'
         -c '  no bgp ebgp-requires-policy'
         -c '  neighbor ROUTERS peer-group'
         -c '  neighbor ROUTERS remote-as external'
         -c '  neighbor ROUTERS default-originate'
         -c '  neighbor net0 interface peer-group ROUTERS'
         -c '  neighbor net1 interface peer-group ROUTERS'
         -c '  address-family ipv4 unicast'
         -c '    redistribute connected'
         -c '  exit-address-family'
         -c '!'
    tor0:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
      - ip link del eth0
      - ip addr add 10.0.0.1/32 dev lo
      - ip addr add 10.0.1.1/24 dev net1
      - ip addr add 10.0.2.1/24 dev net2
      - touch /etc/frr/vtysh.conf
      - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
      - /usr/lib/frr/frrinit.sh start
      - >-
         vtysh -c 'conf t'
         -c 'frr defaults datacenter'
         -c 'router bgp 65010'
         -c '  bgp router-id 10.0.0.1'
         -c '  no bgp ebgp-requires-policy'
         -c '  neighbor ROUTERS peer-group'
         -c '  neighbor ROUTERS remote-as external'
         -c '  neighbor SERVERS peer-group'
         -c '  neighbor SERVERS remote-as internal'
         -c '  neighbor net0 interface peer-group ROUTERS'
         -c '  neighbor 10.0.1.2 peer-group SERVERS'
         -c '  neighbor 10.0.2.2 peer-group SERVERS'
         -c '  address-family ipv4 unicast'
         -c '    redistribute connected'
         -c '  exit-address-family'
         -c '!'
    tor1:
      kind: linux
      image: frrouting/frr:v8.2.2
      labels:
        app: frr
      exec:
      - ip link del eth0
      - ip addr add 10.0.0.2/32 dev lo
      - ip addr add 10.0.3.1/24 dev net1
      - ip addr add 10.0.4.1/24 dev net2
      - touch /etc/frr/vtysh.conf
      - sed -i -e 's/bgpd=no/bgpd=yes/g' /etc/frr/daemons
      - /usr/lib/frr/frrinit.sh start
      - >-
         vtysh -c 'conf t'
         -c 'frr defaults datacenter'
         -c 'router bgp 65011'
         -c '  bgp router-id 10.0.0.2'
         -c '  bgp bestpath as-path multipath-relax'
         -c '  no bgp ebgp-requires-policy'
         -c '  neighbor ROUTERS peer-group'
         -c '  neighbor ROUTERS remote-as external'
         -c '  neighbor SERVERS peer-group'
         -c '  neighbor SERVERS remote-as internal'
         -c '  neighbor net0 interface peer-group ROUTERS'
         -c '  neighbor 10.0.3.2 peer-group SERVERS'
         -c '  neighbor 10.0.4.2 peer-group SERVERS'
         -c '  address-family ipv4 unicast'
         -c '    redistribute connected'
         -c '  exit-address-family'
         -c '!'
    server0:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:control-plane
      exec:
      # Cilium currently doesn't support BGP Unnumbered
      - ip addr add 10.0.1.2/24 dev net0
      # Cilium currently doesn't support importing routes
      - ip route replace default via 10.0.1.1
    server1:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:worker
      exec:
      - ip addr add 10.0.2.2/24 dev net0
      - ip route replace default via 10.0.2.1
    server2:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:worker2
      exec:
      - ip addr add 10.0.3.2/24 dev net0
      - ip route replace default via 10.0.3.1
    server3:
      kind: linux
      image: nicolaka/netshoot:latest
      network-mode: container:worker3
      exec:
      - ip addr add 10.0.4.2/24 dev net0
      - ip route replace default via 10.0.4.1
  links:
  - endpoints: ["router0:net0", "tor0:net0"]
  - endpoints: ["router0:net1", "tor1:net0"]
  - endpoints: ["tor0:net1", "server0:net0"]
  - endpoints: ["tor0:net2", "server1:net0"]
  - endpoints: ["tor1:net1", "server2:net0"]
  - endpoints: ["tor1:net2", "server3:net0"]
root@server:~# containerlab -t topo.yaml deploy
INFO[0000] Containerlab v0.31.1 started                 
INFO[0000] Parsing & checking topology file: topo.yaml  
INFO[0000] Could not read docker config: open /root/.docker/config.json: no such file or directory 
INFO[0000] Pulling docker.io/nicolaka/netshoot:latest Docker image 
INFO[0009] Done pulling docker.io/nicolaka/netshoot:latest 
INFO[0009] Could not read docker config: open /root/.docker/config.json: no such file or directory 
INFO[0009] Pulling docker.io/frrouting/frr:v8.2.2 Docker image 
INFO[0014] Done pulling docker.io/frrouting/frr:v8.2.2  
INFO[0014] Creating lab directory: /root/clab-bgp-cplane-demo 
INFO[0014] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24", IPv6Subnet="2001:172:20:20::/64", MTU="1500" 
INFO[0014] Creating container: "router0"                
INFO[0014] Creating container: "tor0"                   
INFO[0014] Creating container: "tor1"                   
INFO[0014] Creating container: "server2"                
INFO[0014] Creating container: "server1"                
INFO[0014] Creating container: "server0"                
INFO[0014] Creating container: "server3"                
INFO[0016] Creating virtual wire: router0:net0 <--> tor0:net0 
INFO[0016] Creating virtual wire: tor0:net1 <--> server0:net0 
INFO[0016] Creating virtual wire: tor0:net2 <--> server1:net0 
INFO[0016] Creating virtual wire: router0:net1 <--> tor1:net0 
INFO[0016] Creating virtual wire: tor1:net1 <--> server2:net0 
INFO[0016] Creating virtual wire: tor1:net2 <--> server3:net0 
INFO[0017] Adding containerlab host entries to /etc/hosts file 
INFO[0017] Executed command '/usr/lib/frr/frrinit.sh start' on clab-bgp-cplane-demo-tor0. stdout:
Started watchfrr 
INFO[0018] Executed command '/usr/lib/frr/frrinit.sh start' on clab-bgp-cplane-demo-router0. stdout:
Started watchfrr 
INFO[0018] Executed command '/usr/lib/frr/frrinit.sh start' on clab-bgp-cplane-demo-tor1. stdout:
Started watchfrr 
INFO[0018] 🎉 New containerlab version 0.41.2 is available! Release notes: https://containerlab.dev/rn/0.41/#0412
Run 'containerlab version upgrade' to upgrade or go check other installation options at https://containerlab.dev/install/ 
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
| # |             Name             | Container ID |          Image           | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
| 1 | clab-bgp-cplane-demo-router0 | 12fb3507ad7d | frrouting/frr:v8.2.2     | linux | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 2 | clab-bgp-cplane-demo-server0 | 0eb5ae2e685c | nicolaka/netshoot:latest | linux | running | N/A            | N/A                  |
| 3 | clab-bgp-cplane-demo-server1 | 37ddd8bdc385 | nicolaka/netshoot:latest | linux | running | N/A            | N/A                  |
| 4 | clab-bgp-cplane-demo-server2 | 3044addbaf47 | nicolaka/netshoot:latest | linux | running | N/A            | N/A                  |
| 5 | clab-bgp-cplane-demo-server3 | 0ba606507d44 | nicolaka/netshoot:latest | linux | running | N/A            | N/A                  |
| 6 | clab-bgp-cplane-demo-tor0    | a4dd655e4fe2 | frrouting/frr:v8.2.2     | linux | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 7 | clab-bgp-cplane-demo-tor1    | f1f6a4f67834 | frrouting/frr:v8.2.2     | linux | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
+---+------------------------------+--------------+--------------------------+-------+---------+----------------+----------------------+
root@server:~# docker exec -it clab-bgp-cplane-demo-router0 vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.0, local AS number 65000 vrf-id 0
BGP table version 8
RIB entries 15, using 2760 bytes of memory
Peers 2, using 1433 KiB of memory
Peer groups 1, using 64 bytes of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
tor0(net0)      4      65010      65000        46        46        0    0    0 00:01:52            3        9 N/A
tor1(net1)      4      65011      65000        46        45        0    0    0 00:01:51            3        9 N/A

Total number of neighbors 2
root@server:~# docker exec -it clab-bgp-cplane-demo-tor0 vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65010 vrf-id 0
BGP table version 9
RIB entries 15, using 2760 bytes of memory
Peers 3, using 2149 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
router0(net0)   4      65000      65010        98        99        0    0    0 00:04:32            6        9 N/A
10.0.1.2        4          0      65010         0         0        0    0    0    never       Active        0 N/A
10.0.2.2        4          0      65010         0         0        0    0    0    never       Active        0 N/A

Total number of neighbors 3
root@server:~# docker exec -it clab-bgp-cplane-demo-tor1 vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.2, local AS number 65011 vrf-id 0
BGP table version 9
RIB entries 15, using 2760 bytes of memory
Peers 3, using 2149 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor        V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
router0(net0)   4      65000      65011       103       105        0    0    0 00:04:47            6        9 N/A
10.0.3.2        4          0      65011         0         0        0    0    0    never       Active        0 N/A
10.0.4.2        4          0      65011         0         0        0    0    0    never       Active        0 N/A

Total number of neighbors 3
root@server:~# 

root@server:~# cilium install \
    --helm-set ipam.mode=kubernetes \
    --helm-set tunnel=disabled \
    --helm-set ipv4NativeRoutingCIDR="10.0.0.0/8" \
    --helm-set bgpControlPlane.enabled=true \
    --helm-set k8s.requireIPv4PodCIDR=true
🔮 Auto-detected Kubernetes kind: kind
✨ Running "kind" validation checks
✅ Detected kind version "0.17.0"
ℹ️  Using Cilium version 1.12.2
🔮 Auto-detected cluster name: kind-clab-bgp-cplane-demo
🔮 Auto-detected datapath mode: tunnel
🔮 Auto-detected kube-proxy has been installed
ℹ️  helm template --namespace kube-system cilium cilium/cilium --version 1.12.2 --set bgpControlPlane.enabled=true,cluster.id=0,cluster.name=kind-clab-bgp-cplane-demo,encryption.nodeEncryption=false,ipam.mode=kubernetes,ipv4NativeRoutingCIDR=10.0.0.0/8,k8s.requireIPv4PodCIDR=true,kubeProxyReplacement=disabled,operator.replicas=1,serviceAccounts.cilium.name=cilium,serviceAccounts.operator.name=cilium-operator,tunnel=disabled
ℹ️  Storing helm values file in kube-system/cilium-cli-helm-values Secret
🔑 Created CA in secret cilium-ca
🔑 Generating certificates for Hubble...
🚀 Creating Service accounts...
🚀 Creating Cluster roles...
🚀 Creating ConfigMap for Cilium version 1.12.2...
🚀 Creating Agent DaemonSet...
🚀 Creating Operator Deployment...
⌛ Waiting for Cilium to be installed and ready...
✅ Cilium was successfully installed! Run 'cilium status' to view installation health
root@server:~# cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         disabled
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

Deployment        cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet         cilium             Desired: 4, Ready: 4/4, Available: 4/4
Containers:       cilium             Running: 4
                  cilium-operator    Running: 1
Cluster Pods:     3/3 managed by Cilium
Image versions    cilium             quay.io/cilium/cilium:v1.12.2@sha256:986f8b04cfdb35cf714701e58e35da0ee63da2b8a048ab596ccb49de58d5ba36: 4
                  cilium-operator    quay.io/cilium/operator-generic:v1.12.2@sha256:00508f78dae5412161fa40ee30069c2802aef20f7bdd20e91423103ba8c0df6e: 1
root@server:~# cilium config view | grep enable-bgp
enable-bgp-control-plane                   true
root@server:~# 


root@server:~# cat cilium-bgp-peering-policies.yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: rack0
spec:
  nodeSelector:
    matchLabels:
      rack: rack0
  virtualRouters:
  - localASN: 65010
    exportPodCIDR: true
    neighbors:
    - peerAddress: "10.0.0.1/32"
      peerASN: 65010
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeeringPolicy
metadata:
  name: rack1
spec:
  nodeSelector:
    matchLabels:
      rack: rack1
  virtualRouters:
  - localASN: 65011
    exportPodCIDR: true
    neighbors:
    - peerAddress: "10.0.0.2/32"
      peerASN: 65011
root@server:~# kubectl get nodes -l 'rack in (rack0,rack1)'
NAME                                 STATUS   ROLES           AGE   VERSION
clab-bgp-cplane-demo-control-plane   Ready    control-plane   26m   v1.25.3
clab-bgp-cplane-demo-worker          Ready    <none>          26m   v1.25.3
clab-bgp-cplane-demo-worker2         Ready    <none>          26m   v1.25.3
clab-bgp-cplane-demo-worker3         Ready    <none>          26m   v1.25.3
root@server:~# kubectl apply -f cilium-bgp-peering-policies.yaml
ciliumbgppeeringpolicy.cilium.io/rack0 created
ciliumbgppeeringpolicy.cilium.io/rack1 created
root@server:~# docker exec -it clab-bgp-cplane-demo-tor0 vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65010 vrf-id 0
BGP table version 13
RIB entries 23, using 4232 bytes of memory
Peers 3, using 2149 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor                                     V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
router0(net0)                                4      65000      65010       324       326        0    0    0 00:15:42            8       13 N/A
clab-bgp-cplane-demo-control-plane(10.0.1.2) 4      65010      65010        23        29        0    0    0 00:01:02            1       11 N/A
clab-bgp-cplane-demo-worker(10.0.2.2)        4      65010      65010        23        29        0    0    0 00:01:02            1       11 N/A

Total number of neighbors 3
root@server:~# docker exec -it clab-bgp-cplane-demo-tor1 vtysh -c 'show bgp ipv4 summary wide'

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.2, local AS number 65011 vrf-id 0
BGP table version 13
RIB entries 23, using 4232 bytes of memory
Peers 3, using 2149 KiB of memory
Peer groups 2, using 128 bytes of memory

Neighbor                               V         AS    LocalAS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
router0(net0)                          4      65000      65011       325       327        0    0    0 00:15:44            8       13 N/A
clab-bgp-cplane-demo-worker2(10.0.3.2) 4      65011      65011        24        30        0    0    0 00:01:05            1       11 N/A
clab-bgp-cplane-demo-worker3(10.0.4.2) 4      65011      65011        24        30        0    0    0 00:01:06            1       11 N/A

Total number of neighbors 3
root@server:~# kubectl apply -f netshoot-ds.yaml
daemonset.apps/netshoot created
root@server:~# kubectl rollout status ds/netshoot -w
Waiting for daemon set "netshoot" rollout to finish: 0 of 3 updated pods are available...
Waiting for daemon set "netshoot" rollout to finish: 2 of 3 updated pods are available...
daemon set "netshoot" successfully rolled out
root@server:~# SRC_POD=$(kubectl get pods -o wide | grep "cplane-demo-worker " | awk '{ print($1); }')
root@server:~# DST_IP=$(kubectl get pods -o wide | grep worker3 | awk '{ print($6); }')
root@server:~# kubectl exec -it $SRC_POD -- ping $DST_IP
PING 10.1.2.110 (10.1.2.110) 56(84) bytes of data.
64 bytes from 10.1.2.110: icmp_seq=1 ttl=57 time=0.268 ms
64 bytes from 10.1.2.110: icmp_seq=2 ttl=57 time=0.121 ms
64 bytes from 10.1.2.110: icmp_seq=3 ttl=57 time=0.140 ms
64 bytes from 10.1.2.110: icmp_seq=4 ttl=57 time=0.134 ms
64 bytes from 10.1.2.110: icmp_seq=5 ttl=57 time=0.119 ms
64 bytes from 10.1.2.110: icmp_seq=6 ttl=57 time=0.170 ms
64 bytes from 10.1.2.110: icmp_seq=7 ttl=57 time=0.244 ms
64 bytes from 10.1.2.110: icmp_seq=8 ttl=57 time=0.124 ms
64 bytes from 10.1.2.110: icmp_seq=9 ttl=57 time=0.133 ms
64 bytes from 10.1.2.110: icmp_seq=10 ttl=57 time=0.114 ms
64 bytes from 10.1.2.110: icmp_seq=11 ttl=57 time=0.137 ms
64 bytes from 10.1.2.110: icmp_seq=12 ttl=57 time=0.124 ms
64 bytes from 10.1.2.110: icmp_seq=13 ttl=57 time=0.194 ms
64 bytes from 10.1.2.110: icmp_seq=14 ttl=57 time=0.116 ms
64 bytes from 10.1.2.110: icmp_seq=15 ttl=57 time=0.109 ms
64 bytes from 10.1.2.110: icmp_seq=16 ttl=57 time=0.106 ms
64 bytes from 10.1.2.110: icmp_seq=17 ttl=57 time=0.131 ms
64 bytes from 10.1.2.110: icmp_seq=18 ttl=57 time=0.193 ms
64 bytes from 10.1.2.110: icmp_seq=19 ttl=57 time=0.132 ms
64 bytes from 10.1.2.110: icmp_seq=20 ttl=57 time=0.122 ms
64 bytes from 10.1.2.110: icmp_seq=21 ttl=57 time=0.119 ms
64 bytes from 10.1.2.110: icmp_seq=22 ttl=57 time=0.153 ms
64 bytes from 10.1.2.110: icmp_seq=23 ttl=57 time=0.131 ms
64 bytes from 10.1.2.110: icmp_seq=24 ttl=57 time=0.228 ms
64 bytes from 10.1.2.110: icmp_seq=25 ttl=57 time=0.133 ms
64 bytes from 10.1.2.110: icmp_seq=26 ttl=57 time=0.132 ms
64 bytes from 10.1.2.110: icmp_seq=27 ttl=57 time=0.129 ms
64 bytes from 10.1.2.110: icmp_seq=28 ttl=57 time=0.144 ms
64 bytes from 10.1.2.110: icmp_seq=29 ttl=57 time=0.133 ms
64 bytes from 10.1.2.110: icmp_seq=30 ttl=57 time=0.210 ms
64 bytes from 10.1.2.110: icmp_seq=31 ttl=57 time=0.128 ms
64 bytes from 10.1.2.110: icmp_seq=32 ttl=57 time=0.112 ms
64 bytes from 10.1.2.110: icmp_seq=33 ttl=57 time=0.133 ms
64 bytes from 10.1.2.110: icmp_seq=34 ttl=57 time=0.127 ms
64 bytes from 10.1.2.110: icmp_seq=35 ttl=57 time=0.135 ms
64 bytes from 10.1.2.110: icmp_seq=36 ttl=57 time=0.206 ms
64 bytes from 10.1.2.110: icmp_seq=37 ttl=57 time=0.136 ms
64 bytes from 10.1.2.110: icmp_seq=38 ttl=57 time=0.164 ms
64 bytes from 10.1.2.110: icmp_seq=39 ttl=57 time=0.120 ms
64 bytes from 10.1.2.110: icmp_seq=40 ttl=57 time=0.121 ms
64 bytes from 10.1.2.110: icmp_seq=41 ttl=57 time=0.136 ms
64 bytes from 10.1.2.110: icmp_seq=42 ttl=57 time=0.179 ms
64 bytes from 10.1.2.110: icmp_seq=43 ttl=57 time=0.111 ms
64 bytes from 10.1.2.110: icmp_seq=44 ttl=57 time=0.150 ms
64 bytes from 10.1.2.110: icmp_seq=45 ttl=57 time=0.131 ms
64 bytes from 10.1.2.110: icmp_seq=46 ttl=57 time=0.127 ms
^C
--- 10.1.2.110 ping statistics ---
46 packets transmitted, 46 received, 0% packet loss, time 46081ms
rtt min/avg/max/mdev = 0.106/0.144/0.268/0.036 ms
```