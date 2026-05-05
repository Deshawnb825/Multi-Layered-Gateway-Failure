# Asymmetric Routing & Stateful Firewall Incident: Multi-Layered Gateway Failure on pfSense

> **Impact:** Complete loss of internet connectivity from VM + loss of L3 reachability between VM and pfSense gateway
> 
> **Duration:** Active investigation session - full resolution achieved
> 
> **Root Cause:** Four compounding failures across routing, gateway config, and firewall state

---

## Scenario Overview

While modifying pfSense configurations, internet access and L3 communication from a VM to the pfSense router were simultaneously lost. The interface was up, the VLAN was tagged correctly, and L2 appeared healthy - making this a deceptive multi-layer failure that couldn't be resolved by checking any single component in isolation.

This wasn't a single misconfiguration. It was a cascade: a bad static host route on the VM created asymmetric routing, the router's default gateway was pointed at a non-existent host, pfSense built stateful NAT entries around the broken path, and those stale states persisted even after config changes. Fixing any one layer without understanding the others would have resulted in partial recovery at best.

---

## Initial Observations

- **Interface status:** Wired connection present (Ubuntu GNOME shows wired active - no internet indicator)
- **L2 confirmed:** 'ip neigh' shows gateway as 'REACHABLE' - ARP resolved, NIC is functional
- **ip a:** 'ens18' is UP with a valid '10.x.x.x/24' address, BROADCAST/MULTICAST/LOWER_UP flags set
- **ping 8.8.8.8:** 100% packet loss - no internet
- **ping 10.x.x.1 (pfSense gateway):** 100% packet loss - no L3 to router
- **'dig' timed out to local stub resolver**
- **pfSense filter logs:** 'vtnet1.31, match, block, in' - traffic from VM is being blocked on ingress

The presence of L2 reachability with L3 failure immediately ruled out physical/link issues and pointed to routing or policy problems above Layer 2.

---

##Hypothesis Tree
