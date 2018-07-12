# Readme for Demonware

calico-felix was initially forked to disable Reverse Path Filtering (`/proc/sys/net/ipv4/conf/veth*/rp_filter=0`) on the veth's that calico CNI creates.
This was done to support our datacentre's IPVS Layer 4 loadbalancing Direct Server Return (DSR) solution.

Later, it was found that calico-felix was removing routes for calico interfaces from the main routing table that it was not aware of.
This is a problem for routes that were learnt through BIRD/BGP that we want explicitly programmed (and not have it be tampered with).

## route-table sync change
We changed the behaviour so that felix would not tamper with routes that were learnt through the BIRD protocol. Details are captured here in ticket:
https://jira.ihs.demonware.net/browse/DEPCON-777

## rp-filter change
### Why does DSR have to do with this?
By using Direct Server Return, packets destined for the pod is encapsulated using IPIP with the actual Source IP address being the IPVS L4D host.
The packet is then decapsulated within the pod itself, and any return traffic is then destined with the Source IP address as the VIP address (as captured as the Destination IP address of the decapsulated packet).

The veth interface for the Pod is unaware of the VIP address (ie. the veth interface on the host has no VIP address assigned to it). Because of the strict rp_filter mode being enabled, any return traffic for traffic targetting the VIP address is dropped due to not having a valid route/path pointing out the veth interface. 

### Merge changes upstream?
The decision to enable strict rp_filter mode (rp_filter=1) was explicitly made to prevent IPv4 spoofing for policy enforcement reasons.

This was done in favour of simplifying the iptables rules in:
https://github.com/projectcalico/felix/pull/788

```
Calico relies on "strict" RPF checking being enabled to prevent workloads, such as VMs and privileged containers, from spoofing their IP addresses and impersonating other workloads (or hosts).
```

The possiblities of merging this quick patch/change as-is is next to impossible.

However, if we invest some time to making it a tunable felix configuration... then merging it upstream is a possibility since the appetite from the community for a similar solution is there.

### Why are we disabling rp_filter completely and not changing it to loose mode (rp_filter=2)?
Changing it to loose mode will still require us to setup routes for every VIP address on every single host, with not much benefits.

### Disabling rp_filter?! that would open us up to all SORTS of attack!
Unprivileged containers/pods do not have the ability to modify its interfaces which is needed to spoof IP addresses in the first place.

Packet filtering is also done at the edge of the network, which will prevent malicious actors outside of DW from spoofing their way through our network.

So at the end of the day, disabling rp_filter on these veth's will only open up the potential to internal malicious DoS attacks... and who would do that?

## How do I build a calico-node container image with these changes made here in this repo?

First, build the calico/felix container image:
```make calico/felix```

Then, check out:
https://github.com/projectcalico/node

and make the change here to point to local image:
https://github.com/projectcalico/node/blob/573ea10c0541e30bc2a52896d2779df1c6763611/Makefile#L81

change this to false if the calico/felix image is local to your workstation:
https://github.com/projectcalico/node/blob/573ea10c0541e30bc2a52896d2779df1c6763611/Makefile#L102

and then run
```make calico/node```

...

or you could just use ENV variables :)
```FELIX_CONTAINER_NAME=docker.las.... PULL_FELIX=false make calico/node```

and you're set!

## Related Issues
https://github.com/projectcalico/felix/issues/1248
https://dcosjira.atlassian.net/browse/DCOS-265
https://github.com/projectcalico/calicoctl/issues/1082


