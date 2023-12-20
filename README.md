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


