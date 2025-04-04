# Netskope IPSec configuration for BIG-IP

## Names and IP Addressing used in this Example
BIG-IP:
  - 2x VLANs:
    - External:
      - Name: public
      - Self IP: 10.254.101.245 /24
    - Internal:
      - Name: private
      - Self IP: 10.254.201.245 /24
  - Default GW: 10.254.101.254

## 1. IPSec Tunnels
Create a IPsec tunnel configuration per Data Plane (DP). The four examples below are for SYD1, SYD2, MEL1 and MEL3.

### 1.1. Dedicated Self IP for IPSec Tunnel
- Create a Self IP dedicted for the IPSec Tunnel local-address to be used in step 1.5.
```
create net self ipsec_tunnel address 10.245.101.101/24 vlan public_vlan allow-service none
```

### 1.2. IPSec Policies - Interface Mode (Phase 2)
- Create the Phase 2 configuration for each DP. The IKE lifetime is 10 minutes less than the 2 hour ReKey.
```
create net ipsec ipsec-policy ns_syd1_policy { protocol esp mode interface ike-phase2-encrypt-algorithm aes256 ike-phase2-auth-algorithm sha256 ike-phase2-perfect-forward-secrecy modp2048 ike-phase2-lifetime 6600 }
create net ipsec ipsec-policy ns_syd2_policy { protocol esp mode interface ike-phase2-encrypt-algorithm aes256 ike-phase2-auth-algorithm sha256 ike-phase2-perfect-forward-secrecy modp2048 ike-phase2-lifetime 6600 }
create net ipsec ipsec-policy ns_mel1_policy { protocol esp mode interface ike-phase2-encrypt-algorithm aes256 ike-phase2-auth-algorithm sha256 ike-phase2-perfect-forward-secrecy modp2048 ike-phase2-lifetime 6600 }
create net ipsec ipsec-policy ns_mel3_policy { protocol esp mode interface ike-phase2-encrypt-algorithm aes256 ike-phase2-auth-algorithm sha256 ike-phase2-perfect-forward-secrecy modp2048 ike-phase2-lifetime 6600 }
```

### 1.3. IPSec - Traffic Selector
- Traffic Selector is used to as a packet filter that defines what traffic should be handled by a IPsec policy.
- RFC1918 /32 IP addresses are used as the Virtual Server will forward to these IP address as "nodes" in the Pool.
```
create net ipsec traffic-selector ns_syd1_ts { source-address 0.0.0.0/0 destination-address 10.1.1.2/32 ipsec-policy ns_syd1_policy }
create net ipsec traffic-selector ns_syd2_ts { source-address 0.0.0.0/0 destination-address 10.1.2.2/32 ipsec-policy ns_syd2_policy }
create net ipsec traffic-selector ns_mel1_ts { source-address 0.0.0.0/0 destination-address 10.1.3.2/32 ipsec-policy ns_mel1_policy }
create net ipsec traffic-selector ns_mel3_ts { source-address 0.0.0.0/0 destination-address 10.1.4.2/32 ipsec-policy ns_mel3_policy }
````

### 1.4. Tunnel Profiles - IPSec Interface
- Attach the Traffic Selector to the Tunnel Profile
```
create net tunnels ipsec ns_syd1_profile { traffic-selector ns_syd1_ts }
create net tunnels ipsec ns_syd2_profile { traffic-selector ns_syd2_ts }
create net tunnels ipsec ns_mel1_profile { traffic-selector ns_mel1_ts }
create net tunnels ipsec ns_mel3_profile { traffic-selector ns_mel3_ts }
```

### 1.5. IPSec Tunnel
- Create the IPSec Tunnel using the Seld IP as the local-address and the DP IP address as the remote-address
```
create net tunnels tunnel ns_syd1_ipsec { local-address 10.245.101.101 remote-address 163.116.192.38 profile ns_syd1_profile description "Netskope NewEdge - SYD1 Tunnel" }
create net tunnels tunnel ns_syd2_ipsec { local-address 10.245.101.101 remote-address 163.116.211.38 profile ns_syd2_profile description "Netskope NewEdge - SYD2 Tunnel" }
create net tunnels tunnel ns_mel1_ipsec { local-address 10.245.101.101 remote-address 163.116.198.38 profile ns_mel1_profile description "Netskope NewEdge - MEL1 Tunnel" }
create net tunnels tunnel ns_mel3_ipsec { local-address 10.245.101.101 remote-address 163.116.215.38 profile ns_mel3_profile description "Netskope NewEdge - MEL3 Tunnel" }
```

### 1.6. Self IP per Tunnel - Local to the Tunnel
- Create Self IP local to the IPSec Tunnel per Tunnel
- This should be in the same subnet as the destination-address used in step 1.3
```
create net self ns_syd1_self { address 10.1.1.1/30 vlan ns_syd1_ipsec }
create net self ns_syd2_self { address 10.1.2.1/30 vlan ns_syd2_ipsec }
create net self ns_mel1_self { address 10.1.3.1/30 vlan ns_mel1_ipsec }
create net self ns_mel3_self { address 10.1.4.1/30 vlan ns_mel3_ipsec }
```

### 1.7. IPSec IKE Peer (Phase 1)
- Create the IKE Peer. Make sure you change <my-psk> and <my-ipaddress>.
- <my-ipaddress> will be your Internet routable IP. In this setup the VLAN 10.254.101.0/24 is behind a NAT. The NAT IP is used for <my-ipaddress>.
```
create net ipsec ike-peer ns_syd1_ike { remote-address 163.116.192.38 version replace-all-with { v2 } phase1-encrypt-algorithm aes256 phase1-hash-algorithm sha256 phase1-perfect-forward-secrecy modp2048 phase1-auth-method pre-shared-key preshared-key <my-psk> lifetime 84600 traffic-selector replace-all-with { ns_syd1_ts } nat-traversal on my-id-value <my-ipaddress> peers-id-value 163.116.192.38 }
create net ipsec ike-peer ns_syd2_ike { remote-address 163.116.211.38 version replace-all-with { v2 } phase1-encrypt-algorithm aes256 phase1-hash-algorithm sha256 phase1-perfect-forward-secrecy modp2048 phase1-auth-method pre-shared-key preshared-key <my-psk> lifetime 84600 traffic-selector replace-all-with { ns_syd2_ts } nat-traversal on my-id-value <my-ipaddress> peers-id-value 163.116.211.38 }
create net ipsec ike-peer ns_mel1_ike { remote-address 163.116.198.38 version replace-all-with { v2 } phase1-encrypt-algorithm aes256 phase1-hash-algorithm sha256 phase1-perfect-forward-secrecy modp2048 phase1-auth-method pre-shared-key preshared-key <my-psk> lifetime 84600 traffic-selector replace-all-with { ns_mel1_ts } nat-traversal on my-id-value <my-ipaddress> peers-id-value 163.116.198.38 }
create net ipsec ike-peer ns_mel3_ike { remote-address 163.116.215.38 version replace-all-with { v2 } phase1-encrypt-algorithm aes256 phase1-hash-algorithm sha256 phase1-perfect-forward-secrecy modp2048 phase1-auth-method pre-shared-key preshared-key <my-psk> lifetime 84600 traffic-selector replace-all-with { ns_mel3_ts } nat-traversal on my-id-value <my-ipaddress> peers-id-value 163.116.215.38 }
```

## 2. Load Balancing the IPSec Tunnels
### 2.1. Netskope Gateway Node + Monitoring
- Create a node using the address of the remote (Netskope end of the tunnel). This node will be used as a gateway pool member within the load balancing configuration.
```
create ltm node ns_syd1_gw { address 10.1.1.2 monitor none }
create ltm node ns_syd2_gw { address 10.1.2.2 monitor none }
create ltm node ns_mel1_gw { address 10.1.3.2 monitor none }
create ltm node ns_mel3_gw { address 10.1.4.2 monitor none }
```
### 2.2. HTTP Monitor
- Create a HTTP monitor that will send a HTTP Request to a well know site. This HTTP monitor will send the HTTP request over the IPSec Tunnel to NS Proxy, simulating a real Client request. This is just one example of a HTTP monitor.
- I would recommend using more than one HTTP or HTTPS monitor with an “Availability Requirement” of at lease one monitor. This will prevent the IPSec Tunnel from flapping in the event of a single HTTP/S monitor failure.
```
create ltm monitor http ns_http_monitor { defaults-from http interval 15 timeout 46 destination *.http recv "Microsoft NCSI" send "GET /ncsi.txt HTTP/1.1\r\nHost: www.msftncsi.com\r\nUser-Agent: BIG-IP\r\nConnection: Close\r\n" }
```
### 2.3. Gateway Pool
- Create a Gateway Pool using the nodes from step 2.1 and apply the HTTP monitor.
- The Gateway pool can be configured for many different scenarios.
- Below I have included a Failover option OR a Load Balance option using Round-Robin. More advanced Load Balancing options are available depending on the BIG-IP license.

(a) Failover Option
```
create ltm pool ns_gw_pool { members replace-all-with { ns_syd1_gw:0 { priority-group 10 } ns_syd2_gw:0 { priority-group 10 } ns_mel1_gw:0 { priority-group 1 } ns_mel3_gw:0 { priority-group 1 } } min-active-members 2 monitor ns_http_monitor }
```
(b) Load Balance Option
```
create ltm pool ns_gw_pool { members replace-all-with { ns_syd1_gw:0 ns_syd2_gw:0 ns_mel1_gw:0 ns_mel3_gw:0 } load-balancing-mode round-robin monitor ns_http_monitor }
```

## 3. Virtual Servers
### 3.1. Transparent Steering/Forwarding
- To forward the traffic to Netskope NewEdge, create a Virtual Server for TCP:80 and TCP:443.
- I recommend using a custom Fast L4 TCP profile so you can tune the TCP settings. In the example below I have disabled “SYN Cookies” as this is mutually exclusive for transparent proxy configurations.
- Persistence (affinity/sticky sessions) is enabled using Source IP with a default timeout of 180 seconds. The persistence will we honoured across the each Virtual Server using the match-across-virtuals option.
- Source Address and Port Address Translation is disabled.
- Apply the Virtual Server to the correct VLAN specific to your configuration.
```
create ltm profile fastl4 ns_l4_profile { defaults-from fastL4 syn-cookie-enable disabled }
create ltm persistence source-addr ns_source_addr { defaults-from source_addr match-across-virtuals enabled timeout 7200 }
create ltm virtual ns_http_80_vs { destination 0.0.0.0:80 ip-protocol tcp profiles replace-all-with { ns_l4_profile } vlans-enabled vlans replace-all-with { private_vlan } translate-port disabled translate-address disabled pool ns_gw_pool persist replace-all-with { ns_source_addr } description "Forward HTTP to Netskope" }
create ltm virtual ns_https_443_vs { destination 0.0.0.0:443 ip-protocol tcp profiles replace-all-with { ns_l4_profile } vlans-enabled vlans replace-all-with { private_vlan } translate-port disabled translate-address disabled pool ns_gw_pool persist replace-all-with { ns_source_addr } description "Forward HTTPS to Netskope" }
```
### 3.2. Explicit Proxy over Tunnel (EPoT)
- The BIG-IP can also be configured to forward Explicit Proxy over Tunnel (EPoT).
- The Virtual Server configuration is very similar to transparent steering (31.), using the same Fast L4 TCP profile and Persistence configuration.
```
create ltm virtual ns_epot_80_vs { destination 10.245.201.101:80 ip-protocol tcp profiles replace-all-with { ns_l4_profile } vlans-enabled vlans replace-all-with { private_vlan } translate-port disabled translate-address enabled pool ns_gw_pool persist replace-all-with { ns_source_addr } description "Netskope Explicit Proxy" }
```
### 3.3 Cloud Firewall
- If Cloud Firewall is enabled, you have to add additional Virtual Servers for TCP, UDP and ICMP.
```
create ltm virtual ns_all_traffic_vs { destination 0.0.0.0:any ip-protocol any profiles replace-all-with { ns_l4_profile } vlans-enabled vlans replace-all-with { private_vlan } translate-port disabled translate-address disabled pool ns_gw_pool persist replace-all-with { ns_source_addr } description "Forward All Traffic Netskope" }
```
