# Rail Commands to OpenShift

**Goal**: The goal of this document is to take the raw commands that would be run to prep a node and convert them into consumable units for OpenShift via custom resource files that can be applied to the cluster.

## Workflow Sections

- [The Raw Commands](#the-raw-commands)
- [Configuring the HW Offloading](#configuring-the-hw-offloading)
- [Configuring the Physical Interface Attributes](#configuring-the-physical-interface-attributes)
- [Configuring the OVS Bridges ](#configuring-the-ovs-bridges)
- [Configuring the OVS Flows](#configuring-the-ovs-flows)

## The Raw Commands

Below are the raw commands to run on a node that is running OpenvSwitch.  The commands do the following:

 * Enable HW offloading
 * Enable the eswitch setting on a device
 * Enable the number of VFs
 * Enable the mtu to 9216
 * Create OVS bridges per device
 * Configure OVS bridges per device
 * Configure OVS flows

While this would work on a standard package based system like RHEL or Ubuntu we want to take these commands and fit them properly for OpenShift.  Below are the raw commands.  The assumption here is that the udev device naming has been normalized with the naming convention of eth_rail(x).

~~~bash
#Set HW-Offload (once per Node):
$ ovs-vsctl set Open_vSwitch . other_config:hw-offload=true

#Set each PF device to switchdev:
$ devlink dev eswitch set pci/0000:08:00.0 mode switchdev

#Create one VF on each PF device
$ echo 1 > /sys/class/net/eth_rail1/device/sriov_numvfs

#Set the MTU to jumbo frames
$ ip link set dev eth_rail1 mtu 9216

#Create OVS bridges for each PF device:
$ ovs-vsctl add-br br-rail-1
$ ovs-vsctl set bridge br-rail-1 fail-mode=secure
$ ovs-vsctl set bridge br-rail-1 external-ids:rail_uplink=eth_rail1
$ ovs-vsctl set Interface br-rail-1 mtu_request=9216
$ ovs-vsctl add-port br-rail-1 eth_rail1
$ ovs-vsctl set Interface eth_rail1 mtu_request=9216

#Set OVS-Bridge external-ids to tor_ip:
$ ovs-vsctl set bridge br-rail-1 external-ids:rail_peer_ip={{rail_1_tor_ip}}

$ Add IPs to internal bridge port
$ ip addr add rail_1_host_ip/infra_rail_subnet dev br-rail-1
$ ip addr add rail_1_pod_gw_ip dev  br-rail-1

# Admin Up of bridge internal port
$ ip link set dev br-rail-1 up

#Add host OVS flows:
#Note that flows are discarded in case that the ovs switch process is restarted (maybe add to systemd service called when ovs service is restarted)
$ ovs-ofctl add-flow  br-rail-1 “cookie=0x1, arp,arp_tpa= rail_1_host_ip actions=LOCAL”
$ ovs-ofctl add-flow  br-rail-1  “cookie=0x1, arp,arp_tpa= rail_1_pod_gw_ip actions=LOCAL”
$ ovs-ofctl add-flow  br-rail-1  “cookie=0x1, ip,nw_dst= rail_1_host_ip actions=LOCAL”
$ ovs-ofctl add-flow  br-rail-1  “cookie=0x1, ip,nw_dst= rail_1_pod_gw_ip actions=LOCAL”
$ ovs-ofctl add-flow  br-rail-1  “cookie=0x1, arp,arp_tpa=rail_1_tor_ip actions=output:eth_rail1”
$ ovs-ofctl add-flow  br-rail-1  “cookie=0x1, ip,in_port=LOCAL,nw_dst=rail_1_tor_ip/8 actions=output:eth_rail1”
~~~

## Configuring the HW Offloading

Let's look at this in sections.  First we will look at configuring the offload portion.  Manually we would do the following.

~~~bash
#Set HW-Offload (once per Node):
$ ovs-vsctl set Open_vSwitch . other_config:hw-offload=true
~~~

In OpenShift we can achieve the same thing by using the following:

~~~bash
$ cat sriovnetworkpoolconfig-offload.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkPoolConfig
metadata:
  name: sriovnetworkpoolconfig-offload
  namespace: openshift-sriov-network-operator
spec:
  # Add fields here
  ovsHardwareOffloadConfig:
    name: worker
~~~

Create it on the cluster.

~~~bash
$ oc create -f sriovnetworkpoolconfig-offload.yaml 
sriovnetworkpoolconfig.sriovnetwork.openshift.io/sriovnetworkpoolconfig-offload created
~~~

Verify that offload is set to true under other_config.

~~~bash
$ oc debug node/nvd-srv-29.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-29nvidiaengrdu2dcredhatcom-debug-v66nq ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.8
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# ovs-vsctl list open_vswitch
_uuid               : 78300d8c-9134-4ae4-8d47-066a11492e23
bridges             : [6dcbfd65-cb12-44de-8954-efe2256529e4, 9b9b41ba-22c6-4c50-a2cf-0a403816af48]
cur_cfg             : 2733
datapath_types      : [netdev, system]
datapaths           : {system=5154b9d6-3ffa-43f8-9f92-2b7c81fd86fa}
db_version          : "8.8.0"
dpdk_initialized    : false
dpdk_version        : "DPDK 24.11.2"
external_ids        : {hostname=nvd-srv-29.nvidia.eng.rdu2.dc.redhat.com, ovn-bridge-mappings="physnet:br-ex", ovn-bridge-remote-probe-interval="0", ovn-enable-lflow-cache="true", ovn-encap-ip="10.6.135.8", ovn-encap-type=geneve, ovn-is-interconn="true", ovn-memlimit-lflow-cache-kb="1048576", ovn-monitor-all="true", ovn-ofctrl-wait-before-clear="0", ovn-remote="unix:/var/run/ovn/ovnsb_db.sock", ovn-remote-probe-interval="180000", ovn-set-local-ip="true", rundir="/var/run/openvswitch", system-id="bac2184b-58eb-48f2-b517-4bd623f21a30"}
iface_types         : [bareudp, erspan, geneve, gre, gtpu, internal, ip6erspan, ip6gre, lisp, patch, srv6, stt, system, tap, vxlan]
manager_options     : []
next_cfg            : 2733
other_config        : {bundle-idle-timeout="0", hw-offload="true", ovn-chassis-idx-bac2184b-58eb-48f2-b517-4bd623f21a30="", vlan-limit="0"}
ovs_version         : "3.5.2-33.el9fdp"
ssl                 : []
statistics          : {}
system_type         : rhel
system_version      : "9.6"
~~~

## Configuring the Physical Interface Attributes

The next section is setting the values on the physical interface for the rail.

~~~bash
$ cat sriovnetworkpoolconfig-offload.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkPoolConfig
metadata:
  name: sriovnetworkpoolconfig-offload
  namespace: openshift-sriov-network-operator
spec:
  # Add fields here
  ovsHardwareOffloadConfig:
    name: worker
bschmaus@bschmaus-thinkpadp1gen3:~/doca1$ cat rail-example-sriovnetworknodepolicy.yaml
apiVersion: sriovnetwork.openshift.io/v1
kind: SriovNetworkNodePolicy
metadata:
  name: rail-x
  namespace: openshift-sriov-network-operator
spec:
  deviceType: netdevice
  eSwitchMode: "switchdev"
  mtu: 9216
  nicSelector:
    pfNames: ["enp55s0np0"]
  numVfs: 1
  isRdma: true
  linkType: eth
  resourceName: rail-x
  nodeSelector:
    feature.node.kubernetes.io/network-sriov.capable: "true"
~~~

~~~bash
$ oc create -f rail-example-sriovnetworknodepolicy.yaml 
sriovnetworknodepolicy.sriovnetwork.openshift.io/rail-x created
~~~

~~~bash
$ oc debug node/nvd-srv-30.nvidia.eng.rdu2.dc.redhat.com
Starting pod/nvd-srv-30nvidiaengrdu2dcredhatcom-debug-f6thp ...
To use host binaries, run `chroot /host`
Pod IP: 10.6.135.9
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host

sh-5.1# cat /sys/class/net/enp55s0np0/device/sriov_numvfs
1

sh-5.1# ip link |grep enp55
26: enp55s0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 qdisc mq state UP mode DEFAULT group default qlen 1000
29: enp55s0np0_0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DEFAULT group default qlen 1000
    altname enp55s0npf0vf0
31: enp55s0v0: <BROADCAST,MULTICAST> mtu 9216 qdisc noop state DOWN mode DEFAULT group default qlen 1000

sh-5.1# grep PCI_SLOT_NAME /sys/class/net/enp55s0np0/device/uevent
PCI_SLOT_NAME=0000:37:00.0

sh-5.1# devlink dev eswitch show pci/0000:37:00.0
pci/0000:37:00.0: mode switchdev inline-mode none encap-mode basic
~~~

## Configuring the OVS Bridges 

## Configuriung OVS Flows 
