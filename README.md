ご指摘の通り、提供いただいたコンフィグは **EVPN ではなく BGP-L3VPN (VPN-IPv4/v6)** の構成です。SRv6をデータプレーンとし、制御プレーンに伝統的な L3VPN 手法（RFC 9252）を用いた構成として README を作成します。

---

# SRv6 L3VPN (End.DT46) over BGP Unnumbered

Nokia SR OS を使用し、アンダーレイに BGP Unnumbered、オーバーレイに SRv6 L3VPN (End.DT46) を採用した検証環境です。

## 概要

本ラボでは、IPv4 および IPv6 のデュアルスタック通信を単一の SRv6 SID でカプセル化する **End.DT46** の動作を検証します。

* **Underlay**: BGP Unnumbered (IPv6 RA ベース) による等コストマルチパス (ECMP) 構成
* **Overlay**: eBGP Multi-hop による VPN-IPv4/IPv6 ルート交換
* **Encapsulation**: SRv6 (End.DT46 function)
* **VRF**: `vprn 1` における Loopback1 間の相互通信

## トポロジー

![Topology](./image/clab-srv6-dt46.jpg)

## 設定のポイント

1. **BGP Unnumbered & Authentication**:
* アンダーレイは IPv6 Router Advertisement を利用して隣接関係を自動確立。
* 全ての BGP ピアリング（Underlay/Overlay）に MD5 パスワード認証を適用。


2. **SRv6 Locator & End.DT46**:
* 各 TOR で一意な Locator (`1000:0:0:X::/64`) を定義。
* `function end-dt46` を使用し、IPv4/IPv6 両方のトラフィックを VRF `1` へマッピング。


3. **BGP Extended Next-Hop Encoding**:
* `extended-nh-encoding vpn-ipv4 true` により、IPv4 VPN ルートのネクストホップとして IPv6 アドレス（System IP）を利用可能に設定。


4. **Policy-Based Routing Advertisement**:
* `policy-options` を使用し、自身の System IPv6 アドレスと SRv6 Locator プレフィックスのみをアンダーレイ BGP で広報。



---

## ベースコンフィグ

### tor1

<details>  

```text
/configure { policy-options prefix-list "SRv6-Locator" prefix 1000:0:0:1::/64 type exact }
/configure { policy-options prefix-list "System_Lo_IPv6" prefix 1000::1/128 type exact }
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 from prefix-list ["System_Lo_IPv6"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 action action-type accept
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 from prefix-list ["SRv6-Locator"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 action action-type accept
/configure policy-options policy-statement "VRF10-IMPORT" entry 10 action action-type accept
/configure policy-options policy-statement "VRF10-IMPORT" default-action action-type accept
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 ethernet mode network
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure port 1/1/c7 admin-state enable
/configure port 1/1/c7 connector breakout c1-25g
/configure port 1/1/c7/1 admin-state enable
/configure port 1/1/c7/1 ethernet mode network
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure router "Base" autonomous-system 65001
/configure router "Base" router-id 10.0.0.1
/configure router "Base" interface "system" ipv6 address 1000::1 prefix-length 128
/configure router "Base" interface "to-spine" admin-state enable
/configure router "Base" interface "to-spine" port 1/1/c1/1
/configure router "Base" interface "to-spine" ipv4 unnumbered system
/configure { router "Base" interface "to-spine" ipv6 }
/configure router "Base" ipv6 router-advertisement interface "to-spine" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-spine" min-advertisement-interval 10
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" start-label 200000
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" end-label 265534
/configure router "Base" bgp connect-retry 1
/configure router "Base" bgp min-route-advertisement 1
/configure router "Base" bgp initial-send-delay-zero true
/configure router "Base" bgp router-id 10.0.0.1
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp split-horizon true
/configure router "Base" bgp ebgp-default-reject-policy import false
/configure router "Base" bgp ebgp-default-reject-policy export false
/configure router "Base" bgp best-path-selection always-compare-med med-value on
/configure router "Base" bgp best-path-selection always-compare-med strict-as true
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" authentication-key "KrbVPnF6Dg13PM/biw6ErLBhicDE hash2"
/configure router "Base" bgp group "BGP_Unnumbered" type internal
/configure router "Base" bgp group "BGP_Unnumbered" family ipv4 false
/configure router "Base" bgp group "BGP_Unnumbered" family ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" export policy ["EXP-EBGP-LOOPBACK"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-spine" allowed-peer-as ["65000" "65010"]
/configure router "Base" bgp group "SRV6-GROUP" admin-state enable
/configure router "Base" bgp group "SRV6-GROUP" multihop 254
/configure router "Base" bgp group "SRV6-GROUP" next-hop-self true
/configure router "Base" bgp group "SRV6-GROUP" peer-ip-tracking true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" extended-nh-encoding vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv4 true
/configure router "Base" bgp neighbor "1000::2" admin-state enable
/configure router "Base" bgp neighbor "1000::2" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::2" peer-as 65002
/configure router "Base" bgp neighbor "1000::3" admin-state enable
/configure router "Base" bgp neighbor "1000::3" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::3" peer-as 65003
/configure router "Base" bgp segment-routing-v6 family ipv4 ignore-received-srv6-tlvs false
/configure router "Base" bgp segment-routing-v6 family ipv4 add-srv6-tlvs locator-name "tor1-loc"
/configure router "Base" bgp segment-routing-v6 family ipv6 ignore-received-srv6-tlvs true
/configure router "Base" bgp segment-routing-v6 family ipv6 add-srv6-tlvs locator-name "tor1-loc"
/configure router "Base" segment-routing segment-routing-v6 source-address 1000::1
/configure router "Base" segment-routing segment-routing-v6 locator "tor1-loc" admin-state enable
/configure router "Base" segment-routing segment-routing-v6 locator "tor1-loc" block-length 48
/configure router "Base" segment-routing segment-routing-v6 locator "tor1-loc" prefix ip-prefix 1000:0:0:1::/64
/configure router "Base" segment-routing segment-routing-v6 locator "tor1-loc" static-function max-entries 3
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor1-loc" function end 1 srh-mode usp
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor1-loc" function end-dt46 value 2
/configure service vprn "1" admin-state enable
/configure service vprn "1" customer "1"
/configure { service vprn "1" segment-routing-v6 1 locator "tor1-loc" function end-dt46 }
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 admin-state enable
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 route-distinguisher "10.0.0.1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 source-address 1000::1
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target import-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target export-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 instance 1
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 default-locator "tor1-loc"
/configure service vprn "1" interface "Loopback1" admin-state enable
/configure service vprn "1" interface "Loopback1" loopback true
/configure service vprn "1" interface "Loopback1" ipv4 primary address 1.1.1.1
/configure service vprn "1" interface "Loopback1" ipv4 primary prefix-length 32
/configure service vprn "1" interface "Loopback1" ipv6 address 10::1 prefix-length 128
/configure service vprn "1" bgp-vpn-backup ipv4 true
/configure service vprn "1" bgp-vpn-backup ipv6 true
/configure system name "tor1"
/configure system login-control idle-timeout none
/configure system time zone non-standard name "jst"
/configure system time zone non-standard offset "09:00"
/configure system time ntp admin-state enable
/configure { system time ntp server 172.20.20.1 router-instance "management" }

```
</details>  

### tor2

<details>  

```text
/configure { policy-options prefix-list "SRv6-Locator" prefix 1000:0:0:2::/64 type exact }
/configure { policy-options prefix-list "System_Lo_IPv6" prefix 1000::2/128 type exact }
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 from prefix-list ["System_Lo_IPv6"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 action action-type accept
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 from prefix-list ["SRv6-Locator"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 action action-type accept
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 ethernet mode network
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure port 1/1/c7 admin-state enable
/configure port 1/1/c7 connector breakout c1-25g
/configure port 1/1/c7/1 admin-state enable
/configure port 1/1/c7/1 ethernet mode network
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure router "Base" autonomous-system 65002
/configure router "Base" router-id 10.0.0.2
/configure router "Base" interface "system" ipv6 address 1000::2 prefix-length 128
/configure router "Base" interface "to-spine" admin-state enable
/configure router "Base" interface "to-spine" port 1/1/c1/1
/configure router "Base" interface "to-spine" ipv4 unnumbered system
/configure { router "Base" interface "to-spine" ipv6 }
/configure router "Base" ipv6 router-advertisement interface "to-spine" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-spine" min-advertisement-interval 10
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" start-label 200000
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" end-label 265534
/configure router "Base" bgp connect-retry 1
/configure router "Base" bgp min-route-advertisement 1
/configure router "Base" bgp initial-send-delay-zero true
/configure router "Base" bgp router-id 10.0.0.2
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp split-horizon true
/configure router "Base" bgp ebgp-default-reject-policy import false
/configure router "Base" bgp ebgp-default-reject-policy export false
/configure router "Base" bgp best-path-selection always-compare-med med-value on
/configure router "Base" bgp best-path-selection always-compare-med strict-as true
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" authentication-key "KrbVPnF6Dg13PM/biw6ErLBhicDE hash2"
/configure router "Base" bgp group "BGP_Unnumbered" type internal
/configure router "Base" bgp group "BGP_Unnumbered" family ipv4 false
/configure router "Base" bgp group "BGP_Unnumbered" family ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" export policy ["EXP-EBGP-LOOPBACK"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-spine" allowed-peer-as ["65000" "65010"]
/configure router "Base" bgp group "SRV6-GROUP" admin-state enable
/configure router "Base" bgp group "SRV6-GROUP" multihop 254
/configure router "Base" bgp group "SRV6-GROUP" next-hop-self true
/configure router "Base" bgp group "SRV6-GROUP" peer-ip-tracking true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" extended-nh-encoding vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv4 true
/configure router "Base" bgp neighbor "1000::1" admin-state enable
/configure router "Base" bgp neighbor "1000::1" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::1" peer-as 65001
/configure router "Base" bgp neighbor "1000::3" admin-state enable
/configure router "Base" bgp neighbor "1000::3" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::3" peer-as 65003
/configure router "Base" bgp segment-routing-v6 family ipv4 ignore-received-srv6-tlvs false
/configure router "Base" bgp segment-routing-v6 family ipv4 add-srv6-tlvs locator-name "tor2-loc"
/configure router "Base" bgp segment-routing-v6 family ipv6 ignore-received-srv6-tlvs true
/configure router "Base" bgp segment-routing-v6 family ipv6 add-srv6-tlvs locator-name "tor2-loc"
/configure router "Base" segment-routing segment-routing-v6 source-address 1000::2
/configure router "Base" segment-routing segment-routing-v6 locator "tor2-loc" admin-state enable
/configure router "Base" segment-routing segment-routing-v6 locator "tor2-loc" block-length 48
/configure router "Base" segment-routing segment-routing-v6 locator "tor2-loc" prefix ip-prefix 1000:0:0:2::/64
/configure router "Base" segment-routing segment-routing-v6 locator "tor2-loc" static-function max-entries 3
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor2-loc" function end 1 srh-mode usp
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor2-loc" function end-dt46 value 2
/configure service vprn "1" admin-state enable
/configure service vprn "1" customer "1"
/configure { service vprn "1" segment-routing-v6 1 locator "tor2-loc" function end-dt46 }
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 admin-state enable
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 route-distinguisher "10.0.0.2:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 source-address 1000::2
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target import-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target export-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 instance 1
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 default-locator "tor2-loc"
/configure service vprn "1" interface "Loopback1" loopback true
/configure service vprn "1" interface "Loopback1" ipv4 primary address 2.2.2.2
/configure service vprn "1" interface "Loopback1" ipv4 primary prefix-length 32
/configure service vprn "1" interface "Loopback1" ipv6 address 10::2 prefix-length 128
/configure system name "tor2"
/configure system login-control idle-timeout none
/configure system time zone non-standard name "jst"
/configure system time zone non-standard offset "09:00"
/configure system time ntp admin-state enable
/configure { system time ntp server 172.20.20.1 router-instance "management" }

```

</details>  

### tor3

<details>  

```text
/configure { policy-options prefix-list "SRv6-Locator" prefix 1000:0:0:3::/64 type exact }
/configure { policy-options prefix-list "System_Lo_IPv6" prefix 1000::3/128 type exact }
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 from prefix-list ["System_Lo_IPv6"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 to protocol name [bgp]

/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 action action-type accept
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 from prefix-list ["SRv6-Locator"]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 20 action action-type accept
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 ethernet mode network
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure port 1/1/c7 admin-state enable
/configure port 1/1/c7 connector breakout c1-25g
/configure port 1/1/c7/1 admin-state enable
/configure port 1/1/c7/1 ethernet mode network
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c7/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure router "Base" autonomous-system 65003
/configure router "Base" router-id 10.0.0.3
/configure router "Base" interface "system" ipv6 address 1000::3 prefix-length 128
/configure router "Base" interface "to-spine" admin-state enable
/configure router "Base" interface "to-spine" port 1/1/c1/1
/configure router "Base" interface "to-spine" ipv4 unnumbered system
/configure { router "Base" interface "to-spine" ipv6 }
/configure router "Base" ipv6 router-advertisement interface "to-spine" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-spine" min-advertisement-interval 10
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" start-label 200000
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" end-label 265534
/configure router "Base" bgp connect-retry 1
/configure router "Base" bgp min-route-advertisement 1
/configure router "Base" bgp initial-send-delay-zero true
/configure router "Base" bgp router-id 10.0.0.3
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp split-horizon true
/configure router "Base" bgp ebgp-default-reject-policy import false
/configure router "Base" bgp ebgp-default-reject-policy export false
/configure router "Base" bgp best-path-selection always-compare-med med-value on
/configure router "Base" bgp best-path-selection always-compare-med strict-as true
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" authentication-key "KrbVPnF6Dg13PM/biw6ErLBhicDE hash2"
/configure router "Base" bgp group "BGP_Unnumbered" type internal
/configure router "Base" bgp group "BGP_Unnumbered" family ipv4 false
/configure router "Base" bgp group "BGP_Unnumbered" family ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" export policy ["EXP-EBGP-LOOPBACK"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-spine" allowed-peer-as ["65000" "65010"]
/configure router "Base" bgp group "SRV6-GROUP" admin-state enable
/configure router "Base" bgp group "SRV6-GROUP" multihop 254
/configure router "Base" bgp group "SRV6-GROUP" next-hop-self true
/configure router "Base" bgp group "SRV6-GROUP" peer-ip-tracking true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" family vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" extended-nh-encoding vpn-ipv4 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv6 true
/configure router "Base" bgp group "SRV6-GROUP" advertise-ipv6-next-hops vpn-ipv4 true
/configure router "Base" bgp neighbor "1000::1" admin-state enable
/configure router "Base" bgp neighbor "1000::1" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::1" peer-as 65001
/configure router "Base" bgp neighbor "1000::2" admin-state enable
/configure router "Base" bgp neighbor "1000::2" group "SRV6-GROUP"
/configure router "Base" bgp neighbor "1000::2" peer-as 65002
/configure router "Base" bgp segment-routing-v6 family ipv4 ignore-received-srv6-tlvs false
/configure router "Base" bgp segment-routing-v6 family ipv4 add-srv6-tlvs locator-name "tor3-loc"
/configure router "Base" bgp segment-routing-v6 family ipv6 ignore-received-srv6-tlvs true
/configure router "Base" bgp segment-routing-v6 family ipv6 add-srv6-tlvs locator-name "tor3-loc"
/configure router "Base" segment-routing segment-routing-v6 source-address 1000::3
/configure router "Base" segment-routing segment-routing-v6 locator "tor3-loc" admin-state enable
/configure router "Base" segment-routing segment-routing-v6 locator "tor3-loc" block-length 48
/configure router "Base" segment-routing segment-routing-v6 locator "tor3-loc" prefix ip-prefix 1000:0:0:3::/64
/configure router "Base" segment-routing segment-routing-v6 locator "tor3-loc" static-function max-entries 3
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor3-loc" function end 1 srh-mode usp
/configure router "Base" segment-routing segment-routing-v6 base-routing-instance locator "tor3-loc" function end-dt46 value 2
/configure service vprn "1" admin-state enable
/configure service vprn "1" customer "1"
/configure { service vprn "1" segment-routing-v6 1 locator "tor3-loc" function end-dt46 }
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 admin-state enable
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 route-distinguisher "10.0.0.3:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 source-address 1000::3
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target import-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 vrf-target export-community "target:1:1"
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 instance 1
/configure service vprn "1" bgp-ipvpn segment-routing-v6 1 srv6 default-locator "tor3-loc"
/configure service vprn "1" interface "Loopback1" loopback true
/configure service vprn "1" interface "Loopback1" ipv4 primary address 3.3.3.3
/configure service vprn "1" interface "Loopback1" ipv4 primary prefix-length 32
/configure service vprn "1" interface "Loopback1" ipv6 address 10::3 prefix-length 128
/configure system name "tor3"
/configure system login-control idle-timeout none
/configure system time zone non-standard name "jst"
/configure system time zone non-standard offset "09:00"
/configure system time ntp admin-state enable
/configure { system time ntp server 172.20.20.1 router-instance "management" }

```

</details>  

### spine

<details>  

```text
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 from protocol name [direct direct-interface]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 to protocol name [bgp]
/configure policy-options policy-statement "EXP-EBGP-LOOPBACK" entry 10 action action-type accept
/configure port 1/1/c1 admin-state enable
/configure port 1/1/c1 connector breakout c1-100g
/configure port 1/1/c1/1 admin-state enable
/configure port 1/1/c1/1 ethernet mode network
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c1/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure port 1/1/c2 admin-state enable
/configure port 1/1/c2 connector breakout c1-100g
/configure port 1/1/c2/1 admin-state enable
/configure port 1/1/c2/1 ethernet mode network
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c2/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure port 1/1/c3 admin-state enable
/configure port 1/1/c3 connector breakout c1-100g
/configure port 1/1/c3/1 admin-state enable
/configure port 1/1/c3/1 ethernet mode network
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge receive true
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge transmit true
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge tx-tlvs port-desc true
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-name true
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-desc true
/configure port 1/1/c3/1 ethernet lldp dest-mac nearest-bridge tx-tlvs sys-cap true
/configure router "Base" autonomous-system 65010
/configure router "Base" router-id 10.0.0.10
/configure router "Base" interface "system" ipv6 address 1000::10 prefix-length 128
/configure router "Base" interface "to-tor1" admin-state enable
/configure router "Base" interface "to-tor1" port 1/1/c1/1
/configure router "Base" interface "to-tor1" ipv4 unnumbered system
/configure { router "Base" interface "to-tor1" ipv6 }
/configure router "Base" interface "to-tor2" admin-state enable
/configure router "Base" interface "to-tor2" port 1/1/c2/1
/configure router "Base" interface "to-tor2" ipv4 unnumbered system
/configure { router "Base" interface "to-tor2" ipv6 }
/configure router "Base" interface "to-tor3" admin-state enable
/configure router "Base" interface "to-tor3" port 1/1/c3/1
/configure router "Base" interface "to-tor3" ipv4 unnumbered system
/configure { router "Base" interface "to-tor3" ipv6 }
/configure router "Base" ipv6 router-advertisement interface "to-tor1" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-tor1" min-advertisement-interval 10
/configure router "Base" ipv6 router-advertisement interface "to-tor2" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-tor2" min-advertisement-interval 10
/configure router "Base" ipv6 router-advertisement interface "to-tor3" admin-state enable
/configure router "Base" ipv6 router-advertisement interface "to-tor3" min-advertisement-interval 10
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" start-label 200000
/configure router "Base" mpls-labels reserved-label-block "SRV6-16bits-FL" end-label 265534
/configure router "Base" bgp connect-retry 1
/configure router "Base" bgp min-route-advertisement 1
/configure router "Base" bgp initial-send-delay-zero true
/configure router "Base" bgp router-id 10.0.0.10
/configure router "Base" bgp rapid-withdrawal true
/configure router "Base" bgp split-horizon true
/configure router "Base" bgp ebgp-default-reject-policy import false
/configure router "Base" bgp ebgp-default-reject-policy export false
/configure router "Base" bgp best-path-selection always-compare-med med-value on
/configure router "Base" bgp best-path-selection always-compare-med strict-as true
/configure router "Base" bgp rapid-update vpn-ipv4 true
/configure router "Base" bgp rapid-update vpn-ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" authentication-key "KrbVPnF6Dg13PM/biw6ErLBhicDE hash2"
/configure router "Base" bgp group "BGP_Unnumbered" type internal
/configure router "Base" bgp group "BGP_Unnumbered" family ipv4 false
/configure router "Base" bgp group "BGP_Unnumbered" family ipv6 true
/configure router "Base" bgp group "BGP_Unnumbered" export policy ["EXP-EBGP-LOOPBACK"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-tor1" allowed-peer-as ["65001"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-tor2" allowed-peer-as ["65002"]
/configure router "Base" bgp group "BGP_Unnumbered" dynamic-neighbor interface "to-tor3" allowed-peer-as ["65003"]
/configure system name "spine"
/configure system time zone non-standard name "jst"
/configure system time zone non-standard offset "09:00"
/configure system time ntp admin-state enable
/configure { system time ntp server 172.20.20.1 router-instance "management" }

```

</details>  


---
