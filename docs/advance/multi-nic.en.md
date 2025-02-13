# Manage Multiple Interface

Kube-OVN can provide cluster-level IPAM capabilities for other CNI network plugins such as macvlan, vlan, host-device, etc.
Other network plugins can then use the subnet and fixed IP capabilities in Kube-OVN.

Kube-OVN also supports address management when multiple NICs are all of Kube-OVN type.

## Working Principle

By using [Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni), we can add multiple NICs of different networks to a Pod.
However, we still lack the ability to manage the IP addresses of different networks within a cluster.
In Kube-OVN, we have been able to perform advanced IP management such as subnet management, IP reservation, random assignment, fixed assignment, etc. through CRD of Subnet and IP.
Now Kube-OVN extend the subnet to integrate with other different network plugins,
so that other network plugins can also use the IPAM functionality of Kube-OVN.

### Workflow

![work-flow](../static/mult-nic-workflow.png)

The above diagram shows how to manage the IP addresses of other network plugins via Kube-OVN.
The eth0 NIC of the container is connected to the OVN network and the net1 NIC is connected to other CNI networks.
The network definition for the net1 network is taken from the NetworkAttachmentDefinition resource definition in multus-cni.

When a Pod is created, `kube-ovn-controller` will get the Pod add event, find the corresponding Subnet according to the annotation in the Pod,
then manage the address from it, and write the address information assigned to the Pod back to the Pod annotation.

The CNI on the container machine can configure `kube-ovn-cni` as the ipam plugin.
`kube-ovn-cni` will read the Pod annotation and return the address information to the corresponding CNI plugin using the standard format of the CNI protocol.

## Usage

### Install Kube-OVN and Multus

Please refer [One-Click Installation](../start/one-step-install.md) and [Multus how to use](https://github.com/k8snetworkplumbingwg/multus-cni/blob/master/docs/how-to-use.md) to install Kube-OVN and Multus-CNI.

### Provide IPAM for other types of CNI

#### Create NetworkAttachmentDefinition

Here we use macvlan as the second network of the container network and set its ipam to `kube-ovn`:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan
  namespace: default
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "kube-ovn",
        "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
        "provider": "macvlan.default"
      }
    }'
```

- `spec.config.ipam.type`: Need to be set to `kube-ovn` to call the kube-ovn plugin to get the address information.
- `server_socket`: The socket file used for communication to Kube-OVN. The default location is `/run/openvswitch/kube-ovn-daemon.sock`.
- `provider`: The current NetworkAttachmentDefinition's `<name>. <namespace>` , Kube-OVN will use this information to find the corresponding Subnet resource.

### The attached NIC is a Kube-OVN type NIC

At this point, the multiple NICs are all Kube-OVN type NICs.

#### Create NetworkAttachmentDefinition

Set the `provider` suffix to `ovn`:

```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: attachnet
  namespace: default
spec:
  config: '{
      "cniVersion": "0.3.0",
      "type": "kube-ovn",
      "server_socket": "/run/openvswitch/kube-ovn-daemon.sock",
      "provider": "attachnet.default.ovn"
    }'
```

- `spec.config.ipam.type`: Need to be set to `kube-ovn` to call the kube-ovn plugin to get the address information.
- `server_socket`: The socket file used for communication to Kube-OVN. The default location is `/run/openvswitch/kube-ovn-daemon.sock`.
- `provider`: The current NetworkAttachmentDefinition's `<name>. <namespace>` , Kube-OVN will use this information to find the corresponding Subnet resource. It should have the suffix `ovn` here.

### Create a Kube-OVN Subnet

Create a Kube-OVN Subnet, set the corresponding `cidrBlock` and `exclude_ips`, the `provider` should be set to the `<name>. <namespace>` of corresponding NetworkAttachmentDefinition.
For example, to provide additional NICs with macvlan, create a Subnet as follows:

```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: macvlan
spec:
  protocol: IPv4
  provider: macvlan.default
  cidrBlock: 172.17.0.0/16
  gateway: 172.17.0.1
  excludeIps:
  - 172.17.0.0..172.17.0.10
```

`gateway`, `private`, `nat` are only valid for networks with `provider` type ovn, not for attachment networks.

If you are using Kube-OVN as an attached NIC, `provider` should be set to the `<name>. <namespace>.ovn` of the corresponding NetworkAttachmentDefinition, and should end with `ovn` as a suffix.

An example of creating a Subnet with an additional NIC provided by Kube-OVN is as follows:

```yaml
apiVersion: kubeovn.io/v1
kind: Subnet
metadata:
  name: attachnet
spec:
  protocol: IPv4
  provider: attachnet.default.ovn
  cidrBlock: 172.17.0.0/16
  gateway: 172.17.0.1
  excludeIps:
  - 172.17.0.0..172.17.0.10
```

### Create a Pod with Multiple NIC

For Pods with randomly assigned addresses,
simply add the following annotation `k8s.v1.cni.cncf.io/networks`, taking the value `<namespace>/<name>` of the corresponding NetworkAttachmentDefinition.：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: samplepod
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: default/macvlan
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "trap : TERM INT; sleep infinity & wait"]
    image: alpine

```

### Create Pod with a Fixed IP

For Pods with fixed IPs, add `<networkAttachmentName>.<networkAttachmentNamespace>.kubernetes.io/ip_address` annotation：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-ip
  namespace: default
  annotations:
    k8s.v1.cni.cncf.io/networks: default/macvlan
    ovn.kubernetes.io/ip_address: 10.16.0.15
    ovn.kubernetes.io/mac_address: 00:00:00:53:6B:B6
    macvlan.default.kubernetes.io/ip_address: 172.17.0.100
    macvlan.default.kubernetes.io/mac_address: 00:00:00:53:6B:BB
spec:
  containers:
  - name: static-ip
    image: nginx:alpine
```

### Create Workloads with Fixed IPs

For workloads that use ippool, add `<networkAttachmentName>.<networkAttachmentNamespace>.kubernetes.io/ip_pool` annotations:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: default
  name: static-workload
  labels:
    app: static-workload
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-workload
  template:
    metadata:
      labels:
        app: static-workload
      annotations:
        k8s.v1.cni.cncf.io/networks: default/macvlan
        ovn.kubernetes.io/ip_pool: 10.16.0.15,10.16.0.16,10.16.0.17
        macvlan.default.kubernetes.io/ip_pool: 172.17.0.200,172.17.0.201,172.17.0.202
    spec:
      containers:
      - name: static-workload
        image: nginx:alpine
```
