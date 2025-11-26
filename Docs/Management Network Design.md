# The Most Overlooked Yet Critical Part of Network Design: Management Network
In most infrastructure designs, the management (MGMT) network is treated as an afterthought—just another VLAN with a /24 and some ACLs. But this segment is arguably the most critical part of the entire network. When things go wrong—and they eventually will—the ability to access, observe, and control infrastructure determines whether you’re performing a clean recovery or flying blind in a crisis.

A well-designed management network can isolate faults, accelerate recovery, and even prevent incidents from escalating. In contrast, a poorly designed management network often becomes the very bottleneck that prevents remediation, leaving engineers locked out and systems stuck in a failure loop.

## Real-World Lesson: The Facebook Outage
A powerful reminder of the importance of management network design came from the Facebook global outage in 2021. Due to a misconfiguration in backbone routing, all Facebook services—including Instagram, WhatsApp, and internal tools—became unreachable.
Recovering from a routing issue isn’t technically difficult—just a few commands. But the real challenge was that engineers couldn’t reach the systems. Internal DNS, management platforms, and even badge access were down. Teams had to physically travel to remote data centers, wasting precious time just to regain access instead of focusing on troubleshooting and recovery. In large-scale operations, every second of outage equals thousands of dollars lost.
The business impact was massive: revenue loss, reputational damage, and global disruption. And the root cause wasn’t just the routing mistake—it was the lack of a resilient management path to recover from it. A more resilient management path could have significantly reduced recovery time.
A management network is not a luxury—it’s a rescue line. When the main network is down, this is what brings it back. Facebook’s outage reminds us that even the most sophisticated networks can fall apart if engineers can’t reach the devices in time.
Design your management network with failure in mind. Build it as if it’s the only thing that will work when everything else is broken—because one day, as Murphy’s Law says, it will be.
Below are some of my guiding principles when designing management networks. These are not strict rules, but real-world lessons—often learned the hard way. Hopefully, they spark thought and discussion:
## 1. Never, Ever Use the Device-Level ACLs for Management Access Control
Many security teams push for device-level ACLs to restrict management access by source IP. While this may look secure on paper, it becomes a disaster during an outage.
Imagine this: during an outage, the primary management path is down, and you’re scrambling to find any router, switch, or forgotten jumpbox that still has reachability. You finally get a working path, and just as you’re about to cheer—your access is blocked by a hardcoded ACL that denies your source IP. Now you’re locked out of the one path that could have saved the network.
Instead, always put management interfaces in a dedicated management segment. Use a firewall to enforce access control. Unlike scattered device ACLs, firewalls offer:
•	Centralized policy management
•	Logged and auditable access
•	Dynamic source tracking (VPN IP pools, temporary access, etc.)
This approach gives you both security and survivability—the two things you’ll desperately need when things go wrong.
##2. Build a Layer 2 Bridge Between Data Centers
One of the most effective ways to increase the survivability of your management network is to extend Layer 2 connectivity across data centers. This ensures the management segment remains reachable—as long as at least one site is still operational.
Avoid overlay-based technologies like VXLAN or SDN-based L2 stretch, as these often rely on control planes or orchestration systems that may fail during the same outage. Instead, use native, reliable Layer 2 bridging technologies such as VPLS or ELAN, which are simpler, control-plane independent, and support VLAN tagging. That means you can piggyback your management VLANs onto existing inter-DC circuits without needing additional infrastructure.
Example: Dual-Site Management VLAN Failover
In the topology below:
- DC1 uses VLAN 200 for its management network: 10.116.0.0/24
- DC2 uses VLAN 300 for its management network: 10.117.0.0/24
- Both VLANs are extended across the VPLS link
- Each site runs OSPF and HSRP for both VLANs
- Only one side is active for each VLAN under normal conditions

Failover Scenario:
If DC1 suffers an outage and the route to 10.116.0.0/24 is lost:
•	An engineer can log into MGTDC2SW (in DC2)
•	Bring up VLAN 200, enabling the standby HSRP router to become active
•	The default gateway 10.116.0.1 shifts to MGTDC2SW
•	OSPF advertises the 10.116.0.0/24 prefix again—now reachable via DC2
This design allows remote access to be quickly restored without requiring full recovery of the failed data center, ensuring continuity of management visibility.

![Layer 2 mgmt Topo](Image/mgmt_figure1.png)

## 3. Console Servers: Your Last Line of Defense
A console server is a lifesaver during major outages. Choose a model with dual NICs—most enterprise-grade console servers support this—and, if policy allows, consider one with an integrated LTE module for out-of-band access when all internal paths are down.
For example:
- NIC1 connects to VLAN 200 (DC1)
- NIC2 connects to VLAN 300 (DC2)
This setup ensures that even if one data center is completely down, the console server remains reachable via the other. This is especially critical for firewalls, core switches, and management switches—devices that must be reachable to bring the network back up.
Just as important: console servers must not rely solely on TACACS/RADIUS. They should either:
•	Use local credentials by default, or
•	Fail over to local auth gracefully when external AAA systems are unreachable.
The same principle applies to the devices connected to the console server—local credentials via console must be enabled. If TACACS is down and you can’t log in locally, you’ll be forced to go through a password recovery process—which is slow, disruptive, and in some cases, not even supported on certain devices. That’s not the situation you want to face in the middle of a crisis.
When everything else is broken, the console server becomes your last control point—don’t let its access method delay your ability to restore the network and resume business.
## 4. Keep Communication Open
In the Facebook outage, even WhatsApp went down—and that happened to be their internal communication tool. It was the equivalent of going to war and losing your radio on day one. During a crisis, you can’t afford to lose contact with your own team.
Your NOC or incident response team must have a communication path that is independent of the company network. A basic cellphone is often enough, but it must:
•	Be charged and reachable at all times
•	Be physically stored in the NOC
•	Have the phone numbers of all key personnel preloaded
Beyond communication, remember that internal documentation systems may also be down—including password managers, runbooks, and change history. To mitigate this:
•	Use a third-party, cloud-based password vault (e.g. LastPass, 1Password)
•	Store critical credentials and instructions needed for business recovery
•	Ensure that a small number of trusted engineers have offline access or recovery options
And most importantly: print your key management recovery document and store a physical copy in the NOC room. 
Outages are chaotic enough—don’t let lack of communication or inaccessible documentation make things worse.
## 5. Always Have the Key
During a major outage, badge access systems can fail too—especially if they rely on the same internal infrastructure that’s already down. This can leave engineers locked out of the very rooms they need to get into.
Your data center and server rooms must have a fallback access method, such as:
•	A physical key
•	A static keypad code
•	A mechanical override
These access methods should be secure, documented, and known to key personnel, not buried in a ticketing system that might also be offline.
It doesn’t matter how fast you can fix the problem if you’re stuck outside the door.
## 6. Last Resort: The Rescue PC
When all else fails, you need a last-resort access path. One effective method is to maintain a rescue PC, configured specifically for emergency use.
This PC should:
•	Be off the domain, using local credentials
•	Have a static NAT rule preconfigured for direct access from the Internet
•	Be connected to the management network, with direct Layer 2 access to critical devices—no reliance on routing or switching
•	Be powered off or disconnected during normal operations to minimize exposure
If the PC has dual NICs, even better:
•	One NIC connects to the management segment
•	The other connects to an LTE modem or out-of-band network, providing an alternative access path when primary circuits are unavailable
During an outage, the NOC team can power on this PC and enable secure remote access for engineers—bypassing failed infrastructure and accelerating recovery.
It’s your “break-glass” system—silent during uptime, vital during disaster.
## 7. Be Prepared and Be Ready
A design is only as good as its execution under pressure. To ensure your management network will perform when it matters most, you need to test it regularly.
Conduct periodic drills by:
•	Isolating one side of the management network to simulate a partial outage
•	Verifying that failover access through the other site or path still works
•	Powering on the rescue PC to confirm it boots, connects to the management network, and is accessible externally
•	Validating that local credentials work on devices when TACACS or RADIUS is offline
•	Logging into third-party systems—such as password vaults and cloud documentation platforms—to confirm they are accessible and active, and that billing and account status won’t block access during an emergency
These exercises should be scoped to affect only the management network, with no impact on production systems.
Practice now—because when it breaks, it’s too late to realize something was forgotten.
## Conclusion
The management network is not just a support layer—it’s the foundation of your operational control. In a real outage, when users are locked out, systems are down, and dashboards are blank, the management network is what stands between chaos and recovery.
Design it for failure. Test it like your job depends on it—because one day, it will.
If your management path is fragile, undocumented, or untested, you’re not managing your network—you’re hoping it doesn’t break.


