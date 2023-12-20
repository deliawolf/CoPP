# CoPP
Control Plane Policing

CoPP is a Cisco IOS feature that is designed to allow administrators to specify controls over traffic that is directed to a device’s control plane. The goal is to prevent low-priority or unnecessary traffic from overwhelming system resources, which could lead to issues in system performance. CoPP treats the control plane as a separate entity with its own ingress and egress ports.

CoPP allows the implementation of QoS features for the control plane. It is configured with the Cisco IOS Modular QoS CLI (MQC) framework. MQC uses three main concepts: class maps, policy maps, and service policies.

## Control Plane Policing Configuration
### Step 1: Define ACL and create class map.

Step 1: Defining the traffic: In the first step, the interesting traffic is defined in a class map. A common method of defining interesting traffic is to create an access list and reference it in a class map. This example creates a class map for all SNMP, HTTP, HTTPS, and SSH traffic.
```
R1(config)# access-list 100 permit udp any any eq snmp
R1(config)# access-list 100 permit tcp any any eq www
R1(config)# access-list 100 permit tcp any any eq 443
R1(config)# access-list 100 permit tcp any any eq 22
R1(config)# class-map COPP_class
R1(config-cmap)# match access-group 100
```
1. ACL 100 matches SNMP, HTTP, HTTPS, and SSH traffic.
2. Class map COPP_class matches traffic permitted by ACL 100.

### Step 2: Define service policy and policing rates for each class map.

Step 2: Defining a service policy: In this step, the actual QoS policy and associated actions are defined in a policy map. For CoPP, policing is the only valid QoS option. The example shows a policing policy applied to the CoPP class map created in the previous example. The QoS policy is configured to police all SNMP, HTTP, HTTPS, and SSH traffic to 300 kbps and to drop any traffic that exceeds this rate. A second class, class-default, is rate limited to 50 packets per second. The class class-default is special in MQC because it is always automatically placed at the end of every policy map. Match criteria cannot be configured for class-default because it automatically includes an implied match for all packets.
```
R1(config)# policy-map COPP_policy
R1(config-pmap)# class COPP_class
R1(config-pmap-c)# police 300000 conform-action transmit exceed-action drop
R1(config-pmap-c)# class class-default
R1(config-pmap-c)# police rate 50 pps conform-action transmit exceed-action drop
```
1. COPP_class is policed to 300 kbps. Traffic beyond that rate is dropped.
2. class-default is policed to 50 packets per second. Traffic beyond that rate is dropped.

### Step 3: Apply service policy to the control plane.

Step 3: Applying the service policy: The last step is to apply the service policy to the correct interface. Normally, a QoS policy is applied to a physical interface, but in the case of CoPP, it is applied in the control plane configuration mode, as shown in the example:
```
R1(config)# control-plane
R1(config-cp)# service-policy input COPP_policy
```
1. The policy is applied in the input direction to filter packets sent to the control plane.

## Control Plane Policing Verification
```
R1# show access-lists 100
R1# show class-map 
R1# show policy-map
R1# show policy-map control-plane 
```


# COPP cases example

Configure and verify CoPP on R1 to police OSPF, VRRP, ICMP, and Telnet traffic.

## Configure the ACL
```
R1(config)# ip access-list extended COPP-ICMP
R1(config-ext-nacl)# permit icmp any any
R1(config-ext-nacl)# ip access-list extended COPP-TELNET
R1(config-ext-nacl)# permit tcp any any eq telnet
R1(config-ext-nacl)# ip access-list extended COPP-OSPF
R1(config-ext-nacl)# permit ospf any any
R1(config-ext-nacl)# ip access-list extended COPP-VRRP
R1(config-ext-nacl)# permit 112 any host 224.0.0.18
```

The COPP-ICMP ACL will match any ICMP packets sent to the router, the COPP-TELNET ACL will match any Telnet sessions initiated with the router, the COPP-OSPF ACL will match any OSPF packets it receives, and the COPP-VRRP ACL will match any multicast VRRP messages received by R1. Notice the use of protocol number 112 in the VRRP ACL. VRRP messages are neither UDP nor TCP. Instead, VRRP was assigned protocol number 112 by the Internet Assigned Numbers Authority (IANA), in the same way that EIGRP was assigned number 88 and OSPF was assigned number 89. Also, VRRP uses the multicast address 224.0.0.18 to transmit messages on the local segment.

A "deny" statement in the ACL will still forward packets to the control plane but the packets will not be policed. This is useful when certain trusted hosts should have unfettered access to a device’s control plane, for example for Telnet or SSH.

## Configure Class-map
```
R1(config)# class-map COPP-MAP-ICMP
R1(config-cmap)# match access-group name COPP-ICMP
R1(config-cmap)# class-map COPP-MAP-TELNET
R1(config-cmap)# match access-group name COPP-TELNET
R1(config-cmap)# class-map COPP-MAP-OSPF
R1(config-cmap)# match access-group name COPP-OSPF
R1(config-cmap)# class-map COPP-MAP-VRRP
R1(config-cmap)# match access-group name COPP-VRRP
```
Each class map is associated with the appropriate ACL for CoPP traffic classification.

## Configure Policy-map
```
R1(config)# policy-map ROUTER-COPP-POLICY
R1(config-pmap)# class COPP-MAP-ICMP
R1(config-pmap-c)# police 8000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class COPP-MAP-TELNET
R1(config-pmap-c)# police 100000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class COPP-MAP-OSPF  
R1(config-pmap-c)# police 300000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class COPP-MAP-VRRP
R1(config-pmap-c)# police 50000 conform-action transmit exceed-action drop
R1(config-pmap-c)# exit
R1(config-pmap)# class class-default
R1(config-pmap-c)# police 12000 conform-action transmit exceed-action transmit
```
The ICMP class map is policed to 8000 bps to allow easy testing of the CoPP policy in the next steps. Notice that a new class map is included at the end of the policy map. The class-default class map is automatically placed at the end of the policy map. Match criteria cannot be configured for class-default because it automatically includes an implied match for all packets. By the nature of CoPP matching mechanisms, certain traffic types will always end up falling into the default class. This includes traffic such as Layer 2 keepalives and non-IP traffic such as certain IS-IS packets. Because these traffic types are required to maintain the network control plane, class-default should never be policed with both conform and exceed being set with an action of drop. It is also generally considered best practice never to rate-limit the class-default class. It is done here simply to serve as an example.

## Apply the COPP config
```
R1(config)# control-plane
R1(config-cp)# service-policy input ROUTER-COPP-POLICY
R1(config-cp)#
*Jul 17 12:04:29.151: %CP-5-FEATURE: Control-plane Policing feature enabled on Control plane aggregate path
```

## VErify the confi
```
R1# show access-lists
R1# show class-map
R1# show policy-map
R1# show policy-map control-plane
```

## NOTE
notice that there is a substantial number of packets that are being matched to the class-default class map. One possible solution to help identify the source of these packets would be to create a “catch-all” class.
```
R1(config)# ip access-list extended COPP-CATCH-ALL
R1(config-ext-nacl)# permit tcp any any
R1(config-ext-nacl)# permit udp any any
R1(config-ext-nacl)# permit icmp any any
R1(config-ext-nacl)# permit ip any any
R1(config-ext-nacl)# end
```
This ACL could then be matched to a class map called COPP-MAP-CATCH-ALL which could be policed to 50 kbps with a conform action of transmit and an exceed action of drop.

Frequent use of the clear access-list counters and show access-list commands would be useful in tuning all CoPP ACLs. These commands help identify traffic matching each ACL. By being especially aware of traffic matching the new COPP-CATCH-ALL ACL, you be able to identify all previously unmatched traffic. Traffic falling into this class should be investigated to determine if it (1) is legitimate receive-traffic that should have previously been classified (that is, was overlooked in the other ACLs); (2) is attack traffic that should be dropped; or (3) is legitimate transit traffic and acceptable for the new “catch-all” ACL.




