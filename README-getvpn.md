# 🔐 GET-VPN — Tunnelless Group Encryption over a Private WAN

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=flat&logo=cisco&logoColor=white)
![GET-VPN](https://img.shields.io/badge/GET--VPN-GDOI-blue?style=flat)
![COOP](https://img.shields.io/badge/Key%20Servers-COOP%20Redundant-orange?style=flat)
![Status](https://img.shields.io/badge/Status-Phased%20Build-yellow?style=flat)

A **Group Encrypted Transport VPN (GET-VPN)** lab: **tunnelless, any-to-any IPsec** across a private MPLS/WAN, where every site can talk to every other site encrypted — with **no point-to-point tunnels** and the **original IP header preserved**. Two **Key Servers in COOP** distribute one group key to five Group Members over **GDOI**.

The idea: instead of building N² tunnels (DMVPN/FlexVPN), all routers share **one group SA**. Any Group Member encrypts to any other automatically. Ideal for a private WAN that already has routable addressing and wants any-to-any confidentiality.

---

## Why GET-VPN (vs DMVPN / FlexVPN)

| | DMVPN / FlexVPN | GET-VPN |
|---|---|---|
| Model | point-to-point tunnels | **tunnelless** — one shared group key |
| Scaling | tunnels grow with sites | **any-to-any**, no per-pair tunnels |
| IP header | new tunnel header (overlay) | **preserved** (header preservation) |
| Best for | internet / hub-spoke | **private MPLS/WAN**, any-to-any |
| Key distribution | per-tunnel IKE | **GDOI** from a Key Server |
| Redundancy | multiple hubs | **COOP** Key Servers |

> GET-VPN needs a **routable underlay** (the original source/dest stay visible), which is exactly why it fits an **MPLS/private WAN** and **not** the public internet.

---

## Topology

```
        ┌──────────────── GET-VPN group ────────────────┐
        │                                                │
   ┌────┴─────┐  Key-Server-1 (R7)      Key-Server-2 (R8)  ┌────┴─────┐
   │ 10.1.1.0 │──R1(GM)                          R5(GM)──│ 10.1.5.0 │
   └──────────┘   │                                 │    └──────────┘
                  │ 192.168.10.0                     │ 192.168.50.0
              ┌───┴──────────── MPLS / Private-WAN ──┴───┐
              │                  (ISP-R)                  │
              └──┬─────────────┬─────────────┬────────────┘
      192.168.20.0     192.168.30.0    192.168.40.0
          │                 │                │
        R2(GM)            R3(GM)           R4(GM)
          │                 │                │
     ┌────┴─────┐     ┌─────┴────┐    ┌──────┴───┐
     │ 10.1.2.0 │     │ 10.1.3.0 │    │ 10.1.4.0 │
     └──────────┘     └──────────┘    └──────────┘

  KS (R7/R8): hold the policy + group key, distribute via GDOI, run COOP
  GM (R1–R5): register to the KS, receive the group key, encrypt LAN↔LAN
  ISP-R:      the MPLS/private-WAN transport (routes the 192.168.x underlay)
```

**Roles**

| Device | Role | Notes |
|---|---|---|
| R7 (Key-Server-1) | **Primary Key Server** | GDOI group, IKE, IPsec policy, rekey |
| R8 (Key-Server-2) | **Secondary Key Server** | COOP peer of R7 (redundancy) |
| R1–R5 | **Group Members** | register to KS, crypto map GDOI on WAN link |
| ISP-R | MPLS / private-WAN core | underlay routing only (no crypto) |

---

## Addressing

**Underlay (WAN links to ISP-R)**

| Link | Subnet |
|---|---|
| R1 ↔ ISP-R | 192.168.10.0/24 |
| R2 ↔ ISP-R | 192.168.20.0/24 |
| R3 ↔ ISP-R | 192.168.30.0/24 |
| R4 ↔ ISP-R | 192.168.40.0/24 |
| R5 ↔ ISP-R | 192.168.50.0/24 |
| KS1 (R7) ↔ R1 | 192.168.17.0/24 |
| KS2 (R8) ↔ R5 | 192.168.58.0/24 |

**Site LANs (the encrypted "interesting" traffic)**

| Site | LAN |
|---|---|
| R1 | 10.1.1.0/24 |
| R2 | 10.1.2.0/24 |
| R3 | 10.1.3.0/24 |
| R4 | 10.1.4.0/24 |
| R5 | 10.1.5.0/24 |

Underlay routing: **OSPF** (advertise the 192.168.x links and the 10.1.x LANs so every GM and KS is reachable before crypto).

---

## Build Phases

| Phase | What | Status |
|---|---|---|
| 1 | Underlay — ISP-R + all links, OSPF, full reachability | ⬜ |
| 2 | Key Server 1 (R7) — GDOI group, IKE, IPsec, crypto ACL | ⬜ |
| 3 | Group Members (R1–R5) — register to KS, apply GDOI crypto map | ⬜ |
| 4 | Verify — LAN↔LAN traffic encrypted, group SA installed | ⬜ |
| 5 | COOP — Key Server 2 (R8) as redundant KS | ⬜ |

---

## Phase 1 — Underlay (routing before crypto)

Goal: every GM WAN interface and both KS are reachable across ISP-R, and each LAN is advertised — **before any GDOI**.

```cisco
! ===== ISP-R (MPLS/WAN core) — routing only =====
router ospf 1
 network 192.168.0.0 0.0.255.255 area 0
```

```cisco
! ===== Each GM (example R2) — WAN + LAN + OSPF =====
interface Ethernet0/0
 description WAN-TO-ISP
 ip address 192.168.20.2 255.255.255.0
interface Ethernet0/1
 description LAN
 ip address 10.1.2.1 255.255.255.0
router ospf 1
 network 192.168.20.0 0.0.0.255 area 0
 network 10.1.2.0 0.0.0.255 area 0
```

**Verify Phase 1**
```cisco
! From any GM, reach the KS and another GM's WAN + LAN:
ping 192.168.17.7                 ! Key-Server-1 (R7)
ping 10.1.3.1                     ! another site's LAN gateway
show ip route ospf                ! full underlay learned
```

> ⚠️ GET-VPN encrypts traffic that matches the KS policy ACL, but the **underlay must route the original IPs**. Get full reachability first; crypto rides on top.

---

## Phase 2 — Key Server 1 (R7)

Goal: R7 holds the IKE policy, the IPsec transform, the **GDOI group**, and the **policy ACL** that defines which traffic the group encrypts.

```cisco
! ===== KEY SERVER 1 (R7) =====

! Phase-1 IKE (KS ↔ GM registration protection)
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
 lifetime 86400
crypto isakmp key GETKEY123 address 0.0.0.0

! IPsec transform + profile for the DATA plane (group SA)
crypto ipsec transform-set GET-TS esp-aes 256 esp-sha-hmac
 mode tunnel
crypto ipsec profile GET-PROFILE
 set transform-set GET-TS

! RSA key used to sign the rekey messages
crypto key generate rsa modulus 2048 label GETVPN-REKEY

! The policy ACL: WHICH traffic the whole group encrypts (LAN ↔ LAN)
ip access-list extended GET-POLICY
 permit ip 10.1.0.0 0.0.255.255 10.1.0.0 0.0.255.255

! The GDOI group (the heart of GET-VPN)
crypto gdoi group GETVPN
 identity number 1000
 server local
  rekey algorithm aes 256
  rekey retransmit 10 number 2
  rekey authentication mypubkey rsa GETVPN-REKEY
  rekey transport unicast
  sa ipsec 10
   profile GET-PROFILE
   match address ipv4 GET-POLICY
   replay counter window-size 64
  address ipv4 192.168.17.7            ! KS source (this KS)
```

**Verify Phase 2 (after GMs register)**
```cisco
show crypto gdoi                       ! group state, members registered
show crypto gdoi ks members            ! list of registered GMs
show crypto gdoi ks policy             ! the policy pushed to the group
```

> 🧠 **The KS is the brain.** It defines the group key, the encryption policy (the ACL), and periodically **rekeys** the whole group. GMs never define the crypto policy themselves — they download it from the KS. That's what makes GET-VPN "configure the policy once, on the KS."

---

## Phase 3 — Group Members (R1–R5)

Goal: each GM registers to the KS, downloads the group key + policy, and applies the **GDOI crypto map** to its WAN interface. Note how little config a GM needs — the policy lives on the KS.

```cisco
! ===== GROUP MEMBER (example R2) =====

! Phase-1 IKE to reach the KS (must match the KS)
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 5
crypto isakmp key GETKEY123 address 192.168.17.7

! Join the GDOI group and point at the KS
crypto gdoi group GETVPN
 identity number 1000
 server address ipv4 192.168.17.7      ! Key-Server-1

! Apply the group as a crypto map on the WAN interface
crypto map GET-MAP 10 gdoi
 set group GETVPN
interface Ethernet0/0
 crypto map GET-MAP
```

**Verify Phase 3**
```cisco
show crypto gdoi                        ! "Registration status: Registered"
show crypto isakmp sa                   ! IKE to the KS during registration
show crypto gdoi group GETVPN           ! group SA + policy downloaded
```

---

## Phase 4 — Verify Encryption (LAN ↔ LAN)

Goal: prove that traffic between two site LANs is actually encrypted by the group SA — no tunnels involved.

```cisco
! From R2's LAN to R3's LAN (source from the LAN so it matches the policy):
ping 10.1.3.1 source 10.1.2.1

! On R2, confirm the group SA is encrypting:
show crypto ipsec sa                    ! encaps/decaps climbing, group SA
show crypto gdoi gm                     ! GM view: policy + rekey status
```

```
Expect:
  - registration: Registered on every GM
  - LAN↔LAN ping works AND show crypto ipsec sa shows encaps/decaps ✅
  - NO tunnel interfaces — the WAN interface itself encrypts via the group SA
```

> 🔑 **Header preservation:** GET-VPN encrypts the payload but keeps the original source/destination in the IP header, so the MPLS/WAN can still route it normally. That's why GET-VPN needs a routable private underlay and doesn't suit the public internet.

---

## Phase 5 — COOP (Key Server 2, R8)

Goal: add R8 as a **cooperative (COOP)** Key Server so the group survives R7 failing. The two KS sync policy and elect a primary.

```cisco
! ===== KEY SERVER 2 (R8) — mirror of R7 + COOP peer =====
crypto gdoi group GETVPN
 identity number 1000
 server local
  rekey authentication mypubkey rsa GETVPN-REKEY
  rekey transport unicast
  sa ipsec 10
   profile GET-PROFILE
   match address ipv4 GET-POLICY
  address ipv4 192.168.58.8
  redundancy
   local priority 90                    ! R7 higher (100) = primary
   peer address ipv4 192.168.17.7       ! COOP peer = KS1
```
```cisco
! ===== On KS1 (R7) — add the COOP peer back =====
crypto gdoi group GETVPN
 server local
  redundancy
   local priority 100                   ! primary
   peer address ipv4 192.168.58.8       ! COOP peer = KS2
```

**Verify Phase 5**
```cisco
show crypto gdoi ks coop                ! COOP state: primary/secondary, in sync
! Fail the primary (shut R7's interface) → GMs stay up via R8 ✅
```

> 🧠 **COOP = Key Server redundancy.** Both KS hold identical policy; one is primary and drives rekeys. If the primary dies, the secondary takes over so the group keeps getting rekeys and new members can still register — no single point of failure for the key infrastructure.

---

## Troubleshooting Log

_Document each real issue and its fix here as you build — the most valuable part of the writeup._

---

## Key Concepts Reference

- **GET-VPN** — tunnelless group encryption for private WANs; all members share one group SA.
- **GDOI (Group Domain of Interpretation)** — the protocol GMs use to register to the KS and receive the group key/policy.
- **Key Server (KS)** — defines the policy, holds the group key, rekeys the group.
- **Group Member (GM)** — registers to the KS, downloads policy, encrypts/decrypts matching traffic.
- **COOP** — cooperative Key Servers for redundancy (primary/secondary).
- **Header preservation** — the original IP header is kept, so the WAN routes the encrypted packet normally (why GET-VPN suits MPLS, not the internet).
- **Rekey** — the KS periodically pushes a new group key to all members (unicast or multicast).

---

## Disclaimer

Personal home/lab environment for learning. Keys, pre-shared secrets, and addressing are lab-only values — replace them before any real use.
