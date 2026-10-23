# Architecture Réseau Multi-Sites CCNP Enterprise

## SITE 1 : Campus HQ (Routage OSPF, EtherChannel LACP, Spanning-Tree & HSRPv2)

### ASW1 & ASW2

```
enable
configure terminal
hostname ASW1

! Création des VLANs
vlan 10
 name PROD_VLAN
vlan 20
 name WIFI_VLAN
exit

! Port Accès Utilisateur (PC1)
interface FastEthernet 0/10
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

! Trunk & EtherChannel vers ASW2 (Po1)
interface range FastEthernet 0/1 - 2
 switchport mode trunk
 channel-group 1 mode active
exit

! Trunk & EtherChannel vers DSW1 (Po2)
interface range FastEthernet 0/23 - 24
 switchport mode trunk
 channel-group 2 mode active
exit

! Spanning-Tree Rapid-PVST
spanning-tree mode rapid-pvst
end
write
```

```
enable
configure terminal
hostname ASW2

vlan 10
 name PROD_VLAN
vlan 20
 name WIFI_VLAN
exit

! Trunk & EtherChannel vers ASW1 (Po1)
interface range FastEthernet 0/1 - 2
 switchport mode trunk
 channel-group 1 mode active
exit

! Trunk & EtherChannel vers DSW2 (Po3)
interface range FastEthernet 0/23 - 24
 switchport mode trunk
 channel-group 3 mode active
exit

spanning-tree mode rapid-pvst
end
write
```
### DSW1

```
enable
configure terminal
hostname DSW1

ip routing

vlan 10
 name PROD_VLAN
vlan 20
 name WIFI_VLAN
exit

! Spanning-Tree Optimisé
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192

! Trunk & EtherChannel vers DSW2 (Po1)
interface range GigabitEthernet 0/1 - 2
 switchport mode trunk
 channel-group 1 mode active
exit

! Trunk & EtherChannel vers ASW1 (Po2)
interface range FastEthernet 0/23 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 2 mode active
exit

! Interface Routée vers R1_CORE
interface GigabitEthernet 0/0
 no switchport
 ip address 10.10.30.2 255.255.255.0
 no shutdown
exit

! SVI VLAN 10 & HSRPv2 (Active)
interface Vlan 10
 ip address 10.10.10.252 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.254
 standby 10 priority 110
 standby 10 preempt
 standby 10 track GigabitEthernet 0/0 15
 no shutdown
exit

! SVI VLAN 20 & HSRPv2 (Standby)
interface Vlan 20
 ip address 10.10.20.252 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.254
 standby 20 priority 105
 standby 20 preempt
 no shutdown
exit

! Routage OSPF Area 0
router ospf 10
 router-id 2.2.2.2
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.30.0 0.0.0.255 area 0
end
write
```

### DSW2

```
enable
configure terminal
hostname DSW2

ip routing

vlan 10
 name PROD_VLAN
vlan 20
 name WIFI_VLAN
exit

spanning-tree mode rapid-pvst
spanning-tree vlan 20 priority 4096
spanning-tree vlan 10 priority 8192

interface range GigabitEthernet 0/1 - 2
 switchport mode trunk
 channel-group 1 mode active
exit

interface range FastEthernet 0/23 - 24
 switchport trunk encapsulation dot1q
 switchport mode trunk
 channel-group 3 mode active
exit

! Interface Routée vers R1_CORE
interface GigabitEthernet 0/1
 no switchport
 ip address 10.10.40.2 255.255.255.0
 no shutdown
exit

! SVI VLAN 10 & HSRPv2 (Standby)
interface Vlan 10
 ip address 10.10.10.253 255.255.255.0
 standby version 2
 standby 10 ip 10.10.10.254
 standby 10 priority 105
 standby 10 preempt
 no shutdown
exit

! SVI VLAN 20 & HSRPv2 (Active)
interface Vlan 20
 ip address 10.10.20.253 255.255.255.0
 standby version 2
 standby 20 ip 10.10.20.254
 standby 20 priority 110
 standby 20 preempt
 standby 20 track GigabitEthernet 0/1 15
 no shutdown
exit

router ospf 10
 router-id 3.3.3.3
 network 10.10.10.0 0.0.0.255 area 0
 network 10.10.20.0 0.0.0.255 area 0
 network 10.10.40.0 0.0.0.255 area 0
end
write
```
### R1_CORE

```
enable
configure terminal
hostname R1_CORE

interface Loopback0
 ip address 1.1.1.1 255.255.255.255
exit

! Interconnexions internes vers DSW1 et DSW2
interface GigabitEthernet 0/0
 ip address 10.10.30.1 255.255.255.0
 no shutdown
exit

interface GigabitEthernet 0/1
 ip address 10.10.40.1 255.255.255.0
 no shutdown
exit

! Interconnexion WAN eBGP vers R2_HUB
interface GigabitEthernet 0/3
 ip address 10.200.1.2 255.255.255.252
 no shutdown
exit

! Processus OSPF Area 0 (Campus Internal)
router ospf 10
 router-id 1.1.1.1
 network 10.10.30.0 0.0.0.255 area 0
 network 10.10.40.0 0.0.0.255 area 0
 redistribute bgp 65001 subnets
exit

! Processus BGP AS 65001 (Edge)
router bgp 65001
 bgp router-id 11.11.11.11
 neighbor 10.200.1.1 remote-as 200
 redistribute ospf 10
end
write
```

## SITE 2 : Domaine EIGRP (R4_S2_LAN & R5_S2_EDGE - AS 65002)
### R4_S2_LAN

```
enable
configure terminal
hostname R4_S2_LAN

! Connectivité LAN (PC2 / SW_S2)
interface GigabitEthernet 0/1
 description LINK_TO_SW_S2
 ip address 172.16.10.254 255.255.255.0
 no shutdown
exit

! Interconnexion vers R5_S2_EDGE
interface GigabitEthernet 0/0
 description LINK_TO_R5_S2_EDGE
 ip address 10.10.50.2 255.255.255.252
 no shutdown
exit

! Loopbacks de simulation
interface Loopback 20
 ip address 172.16.20.1 255.255.255.0
interface Loopback 30
 ip address 172.16.30.1 255.255.255.0
exit

! Routage EIGRP AS 100
router eigrp 100
 eigrp router-id 4.4.4.4
 network 10.10.50.0 0.0.0.3
 network 172.16.10.0 0.0.0.255
 network 172.16.20.0 0.0.0.255
 network 172.16.30.0 0.0.0.255
 no auto-summary
end
write
```

### R5_S2_EDGE - AS 65002

```
enable
configure terminal
hostname R5_S2_EDGE

! Lien vers R4_S2_LAN
interface GigabitEthernet 0/2
 description LINK_TO_R4_S2_LAN
 ip address 10.10.50.1 255.255.255.252
 no shutdown
exit

! Lien WAN vers R2_HUB (ISP)
interface GigabitEthernet 0/1
 description WAN_TO_R2_HUB
 ip address 10.200.2.2 255.255.255.252
 no shutdown
exit

! Filtre de sécurisation de la redistribution (Prefix-List + Route-Map)
ip prefix-list PERMIT_LAN_S2 seq 10 permit 172.16.0.0/16 ge 24 le 24

route-map FILTER_EIGRP_TO_BGP permit 10
 match ip address prefix-list PERMIT_LAN_S2
exit

! Routage EIGRP AS 100
router eigrp 100
 eigrp router-id 5.5.5.5
 network 10.10.50.0 0.0.0.3
 no auto-summary
 redistribute bgp 65002 metric 100000 10 255 1 1500
exit

! Routage eBGP AS 65002
router bgp 65002
 bgp router-id 12.12.12.12
 neighbor 10.200.2.1 remote-as 200
 redistribute eigrp 100 route-map FILTER_EIGRP_TO_BGP
end
write
```

## SITE 3 : Domaine OSPF Multi-Zones (WAN_CORE, ABRs, R8_S3_LAN - AS 65003)

### WAN_CORE

```
enable
configure terminal
hostname WAN_CORE

! Interconnexion WAN eBGP vers R2_HUB
interface GigabitEthernet 0/2
 description WAN_TO_R2_HUB
 ip address 10.200.3.2 255.255.255.252
 no shutdown
exit

! Interconnexions Backbone OSPF Area 0
interface GigabitEthernet 0/1
 description LINK_TO_R6_S3_ABR1
 ip address 10.10.60.1 255.255.255.252
 no shutdown
exit

interface GigabitEthernet 0/3
 description LINK_TO_R7_S3_ABR2
 ip address 10.10.64.1 255.255.255.252
 no shutdown
exit

! Processus OSPF Area 0
router ospf 10
 router-id 13.13.13.13
 network 10.10.60.0 0.0.0.3 area 0
 network 10.10.64.0 0.0.0.3 area 0
 redistribute bgp 65003 subnets
exit

! Processus eBGP AS 65003
router bgp 65003
 bgp router-id 13.13.13.13
 neighbor 10.200.3.1 remote-as 200
 redistribute ospf 10
end
write
```

### R6_S3_ABR1

```

enable
configure terminal
hostname R6_S3_ABR1

! Interface WAN Area 0
interface GigabitEthernet 0/1
 description LINK_TO_WAN_CORE
 ip address 10.10.60.2 255.255.255.252
 no shutdown
exit

! Interface LAN Area 3
interface GigabitEthernet 0/2
 description LINK_TO_R8_S3_LAN
 ip address 172.17.50.1 255.255.255.252
 no shutdown
exit

router ospf 10
 router-id 6.6.6.6
 network 10.10.60.0 0.0.0.3 area 0
 network 172.17.50.0 0.0.0.3 area 3
 ! Injection de la zone Totally Stubby
 area 3 stub no-summary
 area 3 range 172.17.0.0 255.255.0.0
end
write
```

### R7_S3_ABR2

```
enable
configure terminal
hostname R7_S3_ABR2

! Interface WAN Area 0
interface GigabitEthernet 0/3
 description LINK_TO_WAN_CORE
 ip address 10.10.64.2 255.255.255.252
 no shutdown
exit

! Interface LAN Area 3
interface GigabitEthernet 0/0
 description LINK_TO_R8_S3_LAN
 ip address 172.17.52.1 255.255.255.252
 no shutdown
exit

router ospf 10
 router-id 7.7.7.7
 network 10.10.64.0 0.0.0.3 area 0
 network 172.17.52.0 0.0.0.3 area 3
 area 3 stub no-summary
 area 3 range 172.17.0.0 255.255.0.0
end
write
```

### R8_S3_LAN

```

enable
configure terminal
hostname R8_S3_LAN

! Lien vers ABR1 (Chemin Principal)
interface GigabitEthernet 0/2
 description LINK_TO_ABR1_MAIN
 ip address 172.17.50.2 255.255.255.252
 no shutdown
exit

! Lien vers ABR2 (Chemin de Secours avec coût élevé)
interface GigabitEthernet 0/0
 description LINK_TO_ABR2_BACKUP
 ip address 172.17.52.2 255.255.255.252
 ip ospf cost 50
 no shutdown
exit

! Lien vers LAN Utilisateurs (PC3)
interface GigabitEthernet 0/1
 description LINK_TO_SW_S3
 ip address 172.17.10.254 255.255.255.0
 no shutdown
exit

! Loopbacks de test
interface Loopback 20
 ip address 172.17.20.1 255.255.255.0
interface Loopback 30
 ip address 172.17.30.1 255.255.255.0
exit

! Processus OSPF Area 3
router ospf 10
 router-id 8.8.8.8
 network 172.17.50.0 0.0.0.3 area 3
 network 172.17.52.0 0.0.0.3 area 3
 network 172.17.10.0 0.0.0.255 area 3
 network 172.17.20.0 0.0.0.255 area 3
 network 172.17.30.0 0.0.0.255 area 3
 area 3 stub
end
write
```

### R2_HUB - AS 200

```
enable
configure terminal
hostname R2_HUB

! Interface WAN vers Site 1 (HQ)
interface GigabitEthernet 0/3
 description WAN_TO_SITE1_HQ
 ip address 10.200.1.1 255.255.255.252
 no shutdown
exit

! Interface WAN vers Site 2
interface GigabitEthernet 0/1
 description WAN_TO_SITE2
 ip address 10.200.2.1 255.255.255.252
 no shutdown
exit

! Interface WAN vers Site 3
interface GigabitEthernet 0/2
 description WAN_TO_SITE3
 ip address 10.200.3.1 255.255.255.252
 no shutdown
exit

! Configuration BGP Central Opérateur
router bgp 200
 bgp router-id 10.10.10.10
 neighbor 10.200.1.2 remote-as 65001
 neighbor 10.200.2.2 remote-as 65002
 neighbor 10.200.3.2 remote-as 65003
 
 ! Annonce des sous-réseaux d'interconnexion
 network 10.200.1.0 mask 255.255.255.252
 network 10.200.2.0 mask 255.255.255.252
 network 10.200.3.0 mask 255.255.255.252
end
write
```
