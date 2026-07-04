# Module 09 — Networking

> **Week 9.** The differentiator module. App developers deploy; cloud engineers control *where packets go*. Your goal: private-by-default architectures you can draw, deploy, and — crucially — debug when DNS lies to you.

---

## 1. VNets, subnets, and the rules of the game

- **VNet** = your private address space (RFC1918, e.g. `10.20.0.0/16`); **subnets** partition it. Azure reserves 5 IPs per subnet — a `/24` yields 251 usable.
- **Plan address space like it's forever:** no overlaps with on-prem or peered VNets (peering fails or NAT pain); leave room to grow; a simple scheme (hub `10.0.0.0/16`, spokes `10.1+.0.0/16`) beats cleverness.
- Subnets are the attachment point for: **NSGs**, **route tables (UDRs)**, **service/private endpoints**, and **delegations** (App Service VNet integration, ACA environments, and others *require* a delegated subnet — size it per service docs).
- **NSGs:** stateful allow/deny rules on subnet and/or NIC, evaluated by priority; **service tags** (`Storage`, `AzureFrontDoor.Backend`, `Sql`) instead of IP lists; **ASGs** name your app roles so rules read like intent (`web-asg → sql-asg:1433`). Default rules allow intra-VNet — segment deliberately.
- **Routing:** system routes exist by default; **UDRs** override (classic: `0.0.0.0/0 → Azure Firewall` in the hub for inspected egress). **NAT Gateway** = the answer to SNAT exhaustion and stable egress IPs for a subnet (recall module 3's SNAT pain).
- **Peering:** non-transitive by default — spoke↔spoke traffic goes via hub only if you route it (firewall as the transit hop). Global peering exists.

## 2. Private Endpoint — and the DNS mechanics everyone gets wrong

A **private endpoint (PE)** is a NIC in *your* subnet with a private IP that maps to *one specific instance* of a PaaS service (this SQL server, this storage account's blob sub-resource). Combine with disabling public network access → the service is reachable only inside your network.

**The DNS part (where every real-world PE issue lives):** clients still connect to the public name (`myacct.blob.core.windows.net`). The magic is a CNAME hop: public name → `myacct.privatelink.blob.core.windows.net`. Resolution must then return the private IP *inside* your network and the public IP outside. That requires:

1. A **Private DNS Zone** named `privatelink.blob.core.windows.net` (each service type has its own zone name — SQL is `privatelink.database.windows.net`, Key Vault `privatelink.vaultcore.azure.net`, etc.),
2. An **A record** for the endpoint (auto-managed via the PE's *DNS zone group* — use it), and
3. The zone **linked to every VNet** that must resolve it (and, for on-prem/hybrid, a **DNS forwarder / Azure DNS Private Resolver** so on-prem resolvers can reach Azure DNS).

Debug sequence you'll use forever: `nslookup myacct.blob.core.windows.net` from inside the VNet → expect the privatelink CNAME then a `10.x` answer; a public IP answer means zone missing/unlinked/record absent. **Service endpoints** (the older feature) = keep public IP but source traffic from the subnet over the backbone — simpler, weaker (no on-prem reach, service-level not instance-level); know the difference, prefer PEs.

## 3. The load-balancing ladder — pick by layer and scope

| Service | Layer / scope | Superpowers | Typical seat |
|---|---|---|---|
| **Load Balancer** | L4, regional | Raw TCP/UDP, zone-redundant, HA-ports | In front of VMs/NVAs, internal tiers |
| **Application Gateway (+WAF)** | L7, regional | Path/host routing, TLS termination, **WAF**, private-only ingress option | Regional web ingress, esp. fully private designs |
| **Front Door (+WAF)** | L7, **global** (anycast edge) | CDN caching, TLS at edge, global LB + failover, **Private Link origins** | The internet-facing front of serious web workloads |
| **Traffic Manager** | DNS, global | Cheap DNS steering (perf/priority/weighted) | Legacy/global steering of non-HTTP or mixed estates |

Compositions to recognize: **FD → App Service/ACA** (lock origin to Front Door via `X-Azure-FDID` + service tag); **FD → AppGW → private workloads** (edge + regional WAF/private routing) — justify the double hop or delete it. AppGW when data must stay regional or you need private ingress; FD when audience is global or you want edge caching/failover.

## 4. Hybrid & security appliances (working vocabulary)

- **VPN Gateway** (IPsec over internet; P2S for devs) vs **ExpressRoute** (private circuit; predictable latency, higher bandwidth, compliance). ER + VPN fallback is common.
- **Azure Firewall** (Basic/Standard/**Premium** with TLS inspection/IDPS): the hub's centralized egress/east-west control; DNAT for inbound. Firewall Policy = the manageable rule artifact.
- **Azure Bastion:** browser/native-client RDP/SSH to private VMs — no public IPs on VMs, ever again.
- **DDoS Network Protection** on VNets with public IPs that matter; WAF policies tuned (start Detection, then Prevention).
- **Virtual WAN**: managed global hub-spoke at scale — recognize it as the "many regions/branches" evolution of DIY hub-spoke.

## 5. App-platform networking (tying modules 3–5 together)

- **App Service:** inbound PE + outbound regional VNet integration (delegated subnet) — two separate features, often both.
- **Functions:** Flex/Premium support VNet integration; classic Consumption doesn't (a plan-selection criterion, module 4).
- **Container Apps:** the *environment* is VNet-injected (internal or external); internal env + AppGW/FD in front = private microservices with controlled ingress.
- **PaaS data services:** SQL/Storage/Key Vault/Service Bus Premium/Cosmos → private endpoints + public access disabled = the "private by default" posture reviews expect.
- The **hub-spoke reference**: hub holds firewall, gateways, DNS resolver, Bastion; each workload/env gets a spoke; policy denies public IPs except sanctioned edges (FD/AppGW). Draw this from memory — it's the whiteboard test.

## 6. Decision framework

Internet-facing web → FD Premium (WAF, Private Link origins) unless single-region + regional-only data → AppGW WAF. Internal APIs → private endpoints + internal ingress (ACA internal env / App Service PE), never "IP allowlists on public endpoints." Egress control required → hub firewall + UDR default route + NAT Gateway where scale demands. Every PaaS dependency: PE + private DNS zone + zone link — as a Bicep module you stamp out (module 11), not a portal ritual.

## 7. Lab 09 (~3.5 h — the beefy one)

> [labs.md → Lab 09](../labs.md)

1. Bicep a mini hub-spoke: hub `10.0.0.0/16` (subnets: firewall, bastion, dns), spoke `10.1.0.0/16` (subnets: `app-integration` delegated to App Service, `endpoints`), peered.
2. Move lab-06 SQL + lab-02 Key Vault behind **private endpoints** in `endpoints`; disable public access; create both private DNS zones, link to both VNets, use DNS zone groups.
3. Wire the App Service: VNet-integrate outbound into `app-integration`; prove connectivity; then from Cloud Shell (outside) prove SQL is unreachable publicly.
4. **DNS forensics:** from a small test VM (via Bastion), `nslookup` each service; break it on purpose (unlink one zone), observe the public-IP answer and the resulting auth/connection error, fix, document the debug sequence.
5. Front the app with **Front Door** (or AppGW if you prefer regional): custom probe on `/healthz`, WAF in Detection; lock App Service access restrictions to the FD service tag + your FDID; verify direct hits fail.
6. Tear down carefully (order matters with PEs/links) — keep the Bicep; it's your reusable pattern library.

## 8. Self-check

1. Why do clients keep using the *public* hostname with private endpoints, and what two DNS artifacts make it resolve privately?
2. Service endpoint vs private endpoint — three concrete differences.
3. Your on-prem laptop can't resolve `sql.privatelink.database.windows.net` over VPN. Name the missing component and where it sits.
4. FD vs AppGW: pick for (a) global SaaS, (b) single-region internal LOB app with WAF need, (c) both-together — and why.
5. What is subnet delegation and name two services requiring it.
6. Spoke A can't reach spoke B despite both peered to hub — explain, and the fix.
7. Where does NAT Gateway sit and which two problems from earlier modules does it solve?

## 9. Explain out loud

- Whiteboard-narrate the full packet + DNS story: browser → Front Door → App Service → (VNet) → SQL private endpoint.
- Convince a skeptic that "firewall rule allowing our office IP" is not a private architecture.

## 10. Verify before you build

- Per-service private DNS zone names (tables on Learn — don't guess).
- FD Premium Private Link origin support for your specific origin type.
- ACA environment networking modes & required subnet sizes (changed over time).

*Next: [Module 10 — Observability](10-observability.md).*
