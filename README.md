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

### Tree

**Decision logic:** The firewall was the first suspect because the logs showed explicit block rules. Disabling *pf* entirely via *pfctl -d* was a deliberate isolation step - not a fix attempt. When connectivity didn't restore, it conclusively proved the packet drop was happening for a routing reason, not a policy reason. This is a critical distinction: logs showing "block" can mean firewall policy *or* the firewall matching traffic that was already going to fail due to routing.

---

## Investigation Steps

### Step 1 - Verify L2 is actually up

(ip a & ip neigh)

**Why:** Confirm the problem scope. If L2 were broken, everything downstream would be moot. 'ip neigh confirming 'REACHABLE' means ARP resolved successfully - the NIC and switch path are functional. This scopes the problem to L3 and above.

### Step 2 - Confirm the failure is bidirectional

(ping 8.8.8.8 & 10.x.x.1)

**Why:** Differentiates between "can't reach internet" (NAT/routing issue at egress) vs. "can't even reach the next hop" (local routing or firewall problem). Losing both means the issue is between the VM and its gateway - not an upstream provider problem.

### Step 3 - Check pfSense filter logs

(Logs)

**Why:** pfSense was logging explicit block actions on the VLAN 31 interface for traffic originating from the VM. This looked like a firewall rule misconfiguration - the obvious suspect after a config change. However, the presence of a match/block log entry doesn't confirm the firewall *is* the problem; it confirms the firewall *saw* the traffic and made a decision. The decision could be correct behavior in response to a routing anomaly.

### Step 4 - Eliminate the firewall as root cause

(pfctl -d)

**Why:** Rather than spending time auditing firewall rules, disabling 'pf' entirely is a faster binary test. If connectivity restores → firewall rules are the issue. If it doesn't → the firewall is a symptom or observer, not the cause.

**Result:** No change in connectivity. Firewall eliminated. This is a production-dangerous step that must be reversed immediately, but as a diagnostic it's definitive.

### Step 5 - Packet-level verification with tcpdump

(tcpdump)

**Why:** Confirms whether ICMP requests are actually leaving the VM and arriving at the router interface. Seeing ICMP echo requests with no replies means packets are reaching the destination but responses are not being sent back - pointing to a reply path problem, not a send-path problem. This is the classic asymmetric routing signature.

---

### Step 6 - Interrogate the routing table on pfSense

(route get vm ip)

**Why:** This is the somking gun. 'route get' performs a FIB lookup for a specific destination - it shows you exactly what the kernel will do with a packet destined for that address. The router was forwarding traffic destined for the VM's subnet (10.x.x.x) through 'vtnet0' (WAN) via a gateway of '10.x.x.25' - a host that doesn't exist. The 'Host' and 'Static' flags confirm this was a manually configured, specific-host static route overriding the directly-connected route. This creates asymmetric routing: traffic enters on vtnet1.31, but replies attempt to exit via vtnet0 through a phantom gateway.

---

### Step 7 - Remove the bad static host route

(route delete)

(post-fix output)

**Result:** 'ping 10.x.x.1' works. L3 to gateway restored.

---

### Step 8 - Diagnose remaining internet failure on pfSense

(ping & route get 8.8.8.8)

**Output:**

(output)

**Why:** The default gateway on pfSense was also pointing at '10.x.x.25'. 'arp -a' confirmed the entry for '.25' was '(incomplete) expired' - the MAC was never resolved because the host doesn't exist on that segment. The router had no valid path to the internet.

---

### Step 9 - Verify the phantom gateway via ARP

(arp -a)

**Output:**

(output)

**Why:** An incomplete ARP entry with 'expired' status is definitive proof that '10.x.x.25' has never responded to ARP - it doesn't exist. '10.x.x.1' is the actual gateway on that segment. This was the correct default gateway that pfSense should have been using.

---

### Step 10 - Fix pfSense default gateway

(route delete default)

(ping 8.8.8.8)

---

### Step 11 - Diagnose remaining VM internet failure

(tcpdump)

**Output:**

**Why:** Traffic is reaching the WAN interface but sourcing from '10.x.x.50' instead of the pfSense WAN IP. This means NAT is not being applied - or the wrong NAT rule is matching. The outbound NAT table had an entry referencing the system's internal IP from a stale state built around the broken routing path.

---

### Step 12 - Clear stale firewall state and fix NAT

- Deleted the bad static route from pfSense GUI (Static Routes → Removed blackhole entry pointing to '10.x.x.25')
- Changed default gateway to correct host
- pfSense state table had preserved flows built around the broken path - these needed to be cleared

**Result:** Full end-to-end connectivity restored. 'ping 8.8.8.8' from VM → 0% packet loss.

---

## Key Findings

| Layer | Finding |
| --- | --- |
| L2 | Healthy throughout - this was never the issue |
| L3 (VM→Router) | Bad static host route on pfSense: 10.x.x.50 → 10.x.x.25 via vtnet0' |
| L3 (Router→Internet) | pfSense default gateway pointed at non-existent host '10.x.x.25' |
| Firewall | 'pf' logs showed blcok - but this was a symptom of asymmetric routing, not the cause |
| NAT | Outbound NAT was hitting stale state table entries built around the broken path |
| State Table | pfSense preserved broken flow state even after routing was corrected |

---

## Root Cause Analysis

This was a **four-layer compounding failure**:

**1. Bad static host route (VM→Router path)**

A specific '/32' static host route for '10.x.x.50' was installed on pfSense pointing to '10.x.x.25' via 'vtnet0'. Because host routes have longer prefix matches than network routes, this overrode the directly-connected route for '10.x.x.0/24' via 'vtnet1.31'. Every packet destined for that host was sent out the WAN toward a non-existent next-hop, creating a classic asymmetric routing loop.

**2. Invalid default gateway**

The pfSense default gateway was also configured to use '10.x.x.25' - a host that has never existed on that segment (confirmed via expired ARP). This silently broke all outbound internet routing at the router level, independent of the VM problem.

**3. pfSense state table preserving broken flows**

pfSense's stateful firewall built connection state entries around the broken routing path. Even after the routing was corrected, the state table continued to enforce old forwarding decisions for existing flows - including NAT translations that were sending traffic with the wrong source IP. This is expected stateful firewall behavior, but it's operationally dangerous when the underlying path changes.

**4. Nat applied to wrong source**

Because the state table preserved the broken path, outbound NAT was translating flows to use '10.x.x.50' (an internal IP) instead of the WAN IP, causing internet traffic to be forwarded upstream with a non-routable source address.

**Why this is hard to debug:** Each layer appeared to explain the symptom independently. The firewall logs showed blocks, the routing table looked plausible at first glance. NAT appeared configured. Any single-hypothesis debugger would have spent time on firewall rules, then give up, rather than following the chain through all 4 layers. 

---

## Resolution Steps

1. 'pfctl -d' → confirmed firewall not root cause → re-enabled pf
2. 'route get 10.x.x.50' → identified bad static host route
3. 'route delete 10.x.x.50' → restored correct path via vtnet1.31
4. 'arp -a' → confirmed '10.x.x.25' doesn't exist
5. 'route delete default && route add default 10.x.x.1' → fixed pfSense internet routing
6. Deleted static route from pfSense GUI → removed the blackhole route permanently
7. Cleared pfSense state table → forced NAT to rebuild with correct paths
8. Verified end-to-end: VM 'ping 8.8.8.8' → 0% packet loss

---

## Lessons Learned

**1. GUI-only access is a single point of failure**

The initial loss of GUI access was treated as the problem. It was a symptom. Operators who rely exclusively on web GUIs lose access to the system exactly when they need it most - during a misconfiguration. Console (out-of-band) access is not optional; it's the only reliable recovery path when the management plane is broken.

**2. Filter logs showing "block" don't mean the firewall is the root cuase**

A firewall blocking traffic can be responding correctly to an upstream routing anomaly. 'pfctl -d' is the fastest way to falsify the firewall hypothesis - if disabling pf doesn't fix it, move down the stack.

**3. 'route get <host>' is more diagnostic than 'netstat -rn'**

'netstat -rn' shows the routing table. 'route get' shows what the kernel will *actually do* with a specific destination after longest-prefic-match resolution. These are different. A '/32' host route won't be obvious in a routing table overview but will instantly surface in 'route get' output.

**4. Stateful firewalls will preserve broken behavior after you fix the config**

pfSense (and any stateful device - ASA, iptables conntrack, etc.) builds state around the path that was active when the flow was established. Changing routing or NAT rules doesn't retrocatively fix existing state entries. After a routing change, state tables must be cleared of flows allowed to expire.

**5. An "incomplete" ARP entry is a definitive diagnostic signal**

'(incomplete) expired' in 'arp -a' output means the kernel sent ARP requests that were never answered. This isn't ambiguous - the host doesn't exist at L2. If your default gateway shows incomplete ARP, you have no internet, regardless of what the routing table says.
