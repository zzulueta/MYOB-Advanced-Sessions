# Day 2 – Networking: Slide Deck Script (15 minutes)

---

## Slide 1 – Agenda & Lab Objective *(1 min)*

- What we'll build: a two-tier Azure network from scratch
- Quick architecture diagram (the tree from the lab)
- Outcome: working Load Balancer + App Gateway routing traffic through a secured VNet

---

## Slide 2 – Azure Virtual Networks (VNets) *(2 min)*

- VNets are private, isolated networks inside Azure
- Subnets segment the VNet — each VM/service sits in a subnet
- Key rules: no overlapping CIDR ranges, five addresses reserved per subnet
- Show: `CoreServicesVnet (10.20.0.0/16)` → `AppVnet (10.60.0.0/16)`

---

## Slide 3 – Network Security Groups (NSGs) *(2 min)*

- Stateful layer 4 firewall applied to a subnet or NIC
- Rules evaluated by priority (lowest number wins)
- Default rules: AllowVNetInBound, DenyAllInBound
- Demo point: `DatabaseSubnet` only allows TCP 1433 from `10.60.0.0/16` — everything else denied

---

## Slide 4 – Azure DNS: Public vs Private Zones *(2 min)*

- **Public zone** (`adventuretravel.com`): resolves from the internet
- **Private zone** (`private.adventuretravel.com`): resolves only within linked VNets
- **Auto-registration**: VMs automatically get DNS records when they join a linked VNet
- **Resolver link**: AppVnet VMs can look up records without registering their own

---

## Slide 5 – VNet Peering *(2 min)*

- Connects two VNets so traffic flows privately (no internet, no gateway)
- Peering is non-transitive — A↔B and B↔C doesn't mean A↔C
- Bidirectional: creates two peering objects (one on each VNet)
- Lab: `CoreServicesVnet ↔ AppVnet` lets vm0/vm1 reach `DatabaseSubnet`

---

## Slide 6 – Azure Load Balancer *(2 min)*

- Layer 4 (TCP/UDP) load balancer
- Components: Frontend IP → Load-balancing rule → Backend pool → Health probe
- Standard SKU supports availability zones and HTTPS probes
- Lab: distributes HTTP traffic across vm0 and vm1 (round-robin)

---

## Slide 7 – Application Gateway *(2 min)*

- Layer 7 (HTTP/HTTPS) load balancer — aware of URLs, headers, cookies
- **Path-based routing**: send `/image/*` to one backend, `/video/*` to another
- Lives in its own dedicated subnet (`AppGwSubnet`)
- Lab: `/image/*` → vm0, `/video/*` → vm1, default → both

---

## Slide 8 – Architecture Recap & Lab Kick-off *(1 min)*

- Show the full architecture diagram one more time
- Call out the 6 tasks and estimated time (~90 min)
- Note: Tasks 1–3 are portal-only; Task 4 uses Cloud Shell; Tasks 5–6 back to portal
