# Multiple routes on sdwan config
Question: when setting up sdwan routes on Cisco SDWAN Edge routers, would having multiple routes such as: ip route 0.0.0.0 0.0.0.0 x.x.x.x for each WAN facing interface be an issue?

###### ChatGPT said:

Yes, having **multiple `ip route 0.0.0.0 0.0.0.0` (default routes)** pointing to different WAN interfaces **on Cisco SD-WAN (Viptela)** edge routers is not inherently an issue, **if configured properly**. In fact, it's a **common practice** in SD-WAN to support **multiple WAN transport paths**, such as internet, MPLS, and LTE.

However, **how these routes are interpreted and used** depends on **how SD-WAN policies and configurations are set up**, particularly:

---

### ✅ When it's **NOT** an issue:

If your configuration is **intentional and controlled**, then multiple default routes are fine:

1. **Equal-cost routes**:
    
    - If multiple `0.0.0.0/0` routes have the **same administrative distance and metric**, they will be treated as **equal-cost multipath (ECMP)** routes.
        
    - SD-WAN can load-balance traffic across these paths (depending on the policy).
        
2. **SD-WAN Transport Interface Configuration**:
    
    - In Cisco SD-WAN, WAN interfaces are typically configured under `vpn 0`, and control connections (to vSmart, vBond, vManage) are built over these interfaces.
        
    - Default routes may be needed for the device to **reach the control plane (DTLS/TLS tunnels)** over each transport.
        
3. **TLOC & BFD Probes**:
    
    - Cisco SD-WAN doesn't rely only on static routing. It uses **TLOCs** (Transport Locators) and **BFD** to monitor paths and make real-time decisions.
        
    - Having multiple default routes enables **TLOCs to register** via each WAN link.
        

---

### ⚠️ When it **CAN** be an issue:

1. **Incorrect priorities / Administrative Distance**:
    
    - If one default route is meant to be primary but you don't adjust the **AD (administrative distance)** or metric, the router may use the "wrong" route.
        
2. **Lack of Policy Control**:
    
    - Without **centralized or localized policy**, traffic could go over unintended links, especially if **data-policy or app-route-policy** is not applied.
        
3. **Overlapping or Conflicting Static Routes**:
    
    - If you're using **manual static routes** instead of relying on SD-WAN's automatic TLOC-based routing, and they conflict or loop, it could break connectivity.
        
4. **NAT/Firewall Asymmetry**:
    
    - If a session enters via one WAN and exits via another, it could break due to **NAT or firewall state tracking**.
        

---

### ✅ Best Practices:

- Use **SD-WAN centralized policies** to control **traffic direction, failover, and application-aware routing**.
    
- Let **OMP (Overlay Management Protocol)** handle most of the routing through TLOC advertisements and route propagation.
    
- Only configure multiple static default routes if you:
    
    - Want to provide failover or ECMP.
        
    - Have configured **tracking** or **BFD-based path monitoring**.
        
    - Understand how SD-WAN fabric handles **control and data plane routing**.
        

---

### 📌 Example:

`ip route 0.0.0.0 0.0.0.0 192.0.2.1 ip route 0.0.0.0 0.0.0.0 198.51.100.1`

This would be valid if `192.0.2.1` and `198.51.100.1` are gateways on two different WAN interfaces (e.g., internet and MPLS), and your policies or TLOCs support them.

---

### Final Note:

In **Cisco SD-WAN**, the **overlay fabric** handles most routing dynamically via **OMP**, and static default routes are often just used to reach **underlay next hops** (for tunnel initiation). So, multiple static default routes are **fine and even recommended**—as long as you understand **how the SD-WAN handles transport routing and policy application**.

If you're seeing unexpected behavior, consider using `show ip route`, `show control connections`, and `show sdwan bfd sessions` to diagnose path selection.

# Onboard Physical Router
in vManage, I have a list of WAN Edges each with their own Chassis Number. These are all Virtualized edges yes? If so, how would I join a physical edge with its own Chassis and Token with my SDWAN, and could I safely reset it to factory defaults?

###### ChatGPT said:

Yes, let’s break this down step-by-step.

---

### ✅ **1. vManage and Chassis Numbers – Are these virtualized WAN Edges?**

In **vManage**, the Chassis Number identifies each WAN Edge device. These could be:

- **vEdge Cloud (virtualized WAN Edges)** – most commonly used in virtual environments (e.g., AWS, Azure, VMware).
    
- **vEdge/cEdge Physical devices** – physical routers (like ISR, ASR, or ENCS running SD-WAN).
    

> So, **just having a Chassis Number doesn't necessarily mean the device is virtual.** You’ll need to check the **model number** or **platform type** in vManage to confirm whether it's a **virtual** or **physical** edge.

---

### ✅ **2. How do I onboard a **physical** WAN Edge device into my SD-WAN fabric?**

To join a physical WAN Edge to your SD-WAN network, you typically use a process called **Controller/Plug-and-Play onboarding**. Here's how:

#### 🔷 Step-by-Step – Onboarding a Physical WAN Edge:

1. **Obtain the Chassis Number & Serial Number**:
    
    - These are on a sticker on the physical device, or can be seen in the boot log/CLI.
        
    - You'll need them to generate a **Bootstrap Configuration / Token**.
        
2. **Add the Device to vManage**:
    
    - Go to `Configuration > Devices > WAN Edge List`.
        
    - Click **"Add WAN Edge"**.
        
    - Enter the Chassis Number and Serial Number of the physical device.
        
    - vManage will generate a **.viptela license file** or **Bootstrap configuration**.
        
3. **Install the Token / Bootstrap Config on the Device**:
    
    - Console into the physical device.
        
    - Apply the token using a command like:
        
        `request platform software sdwan vedge-cloud activate chassis-number <chassis#> token <token>`
        
    - Or copy over the **Bootstrap config file** to the device via USB or SCP.
        
4. **Connectivity**:
    
    - The device must be able to reach **vBond**, **vSmart**, and **vManage**.
        
    - After successful validation, it will appear as a **"validated"** device in vManage and you can assign it a full configuration template.
        

---

### ✅ **3. Can I safely reset the physical WAN Edge to factory defaults?**

**Yes, but consider these first:**

#### 🔹 Use `request software reset` for Factory Reset:

- For **vEdge devices**, use:
    
    `request system erase`
    
    or
    
    `request software reset`
    
- For **cEdge (IOS XE SD-WAN)** devices, use:
    
    `write erase reload`
    

#### 🔹 Things to consider before resetting:

- **Back up any existing config** (if needed).
    
- Make sure you still have access to the **Bootstrap token or configuration**.
    
- A factory reset will erase:
    
    - All configurations
        
    - Certificates
        
    - Device identity and logs
        

After reset, the device will be in a **"ZTP" state** and can be re-onboarded as if new.

---

### 📝 Summary:

|Question|Answer|
|---|---|
|Are devices in vManage with Chassis Numbers virtual?|Not always – check the model type.|
|How to onboard a physical WAN Edge?|Add to WAN Edge List → Apply Token or Bootstrap → Ensure connectivity to controllers.|
|Can I safely reset a physical WAN Edge?|Yes, with proper backups and token/config in hand.|

---

If you'd like, I can walk you through generating a token or verifying the device type in your vManage UI. Just let me know what you see under `Configuration > Devices`.
# BWAN!
what is BWAN?

###### ChatGPT said:

The acronym **BWAN** can have several different meanings depending on the context. Some of the most common ones are:

- “Broadband Wireless Access Network” — referring to a wireless network architecture for broadband access. [Acronym Finder+2Acronym Finder+2](https://www.acronymfinder.com/Broadband-Wireless-Access-Network-%28BWAN%29.html?utm_source=chatgpt.com)
    
- “Blog Writers and Artists Network” — as in a community or network of creators. [Acronym Finder+1](https://www.acronymfinder.com/Blog-Writers-and-Artists-Network-%28BWAN%29.html?utm_source=chatgpt.com)
    
- “Bureau Wide Area Network” — another more generic IT/networking expansion. [The Free Dictionary+1](https://acronyms.thefreedictionary.com/BWAN?utm_source=chatgpt.com)
    

If you tell me **which context** you encountered “BWAN” (e.g., telecommunications, a blog/forum, corporate IT, etc.), I can check and give **the most likely meaning** for that scenario.

##### You said:

Borderless WAN?

###### ChatGPT said:

Ah — yes, in that context, **BWAN** often refers to **Borderless WAN**, a concept or solution architecture used in enterprise networking.

Here’s what **Borderless WAN (BWAN)** means:

### 🔍 Definition

**Borderless WAN** is a **network architecture approach** that removes traditional boundaries between data centers, branch offices, cloud services, and remote users. It allows secure, seamless connectivity across multiple network domains — regardless of **location, device, or transport medium (MPLS, Internet, 5G, etc.)**.

### 🧠 Core Idea

It’s part of Cisco’s **Borderless Network Architecture**, which aims to give users and devices secure, reliable access to network resources “anytime, anywhere.”

### ⚙️ Key Features

- **Unified connectivity:** Integrates multiple WAN technologies (MPLS, broadband, LTE/5G, etc.) under a single management layer.
    
- **Security built-in:** Uses secure tunnels, firewalls, and policy enforcement across all edges.
    
- **Application optimization:** Prioritizes critical applications and ensures quality of service (QoS).
    
- **Cloud readiness:** Designed for hybrid and multicloud environments.
    
- **Centralized management:** Often paired with SD-WAN controllers for policy and analytics.
    

### 🏢 Example

A company with offices worldwide can use a **Borderless WAN** to ensure employees can access corporate apps whether they’re on-site, at home, or connected via cloud — all while maintaining consistent performance and security.

So yes — **BWAN = Borderless Wide Area Network**, and it’s essentially a **modern, software-defined evolution of traditional WANs**, built for the cloud era.

Would you like me to compare **BWAN vs SD-WAN**? They’re closely related but not identical.

##### You said:

so is BWAN a subset of SD WAN?

###### ChatGPT said:

Excellent question — and it’s a subtle one. The short answer is:

> **BWAN (Borderless WAN)** isn’t exactly a _subset_ of SD-WAN, but rather an **architectural concept** that _includes_ or _can be implemented using_ SD-WAN technologies.

Let’s unpack that carefully.

---

### 🧭 1. **Conceptual Scope**

|Aspect|**BWAN (Borderless WAN)**|**SD-WAN (Software-Defined WAN)**|
|---|---|---|
|**Definition**|A _design philosophy_ from Cisco’s Borderless Network Architecture that focuses on secure, seamless access across network boundaries.|A _technology framework_ that uses software-defined networking (SDN) principles to manage and optimize WAN connections.|
|**Purpose**|Eliminate “borders” between users, devices, and resources (data center, cloud, branch).|Simplify WAN management and improve performance using centralized control and dynamic path selection.|
|**Scope**|Broader — includes not just connectivity but also identity, policy, security, and mobility.|More focused — on WAN traffic routing, optimization, and orchestration.|

---

### ⚙️ 2. **Relationship**

You can think of **SD-WAN as an _enabler_ of the Borderless WAN vision**.

- A **BWAN deployment** might _use_ SD-WAN technology as its underlying transport and control layer.
    
- But **BWAN also covers other areas** like end-to-end security, policy enforcement, cloud access, and mobility services — all parts of Cisco’s larger _Borderless Network_ architecture (which also included Borderless Campus, Borderless Data Center, etc.).
    

---

### 🧩 3. **Analogy**

If the **Borderless Network** is the _strategy_,  
and the **Borderless WAN** is the _architectural goal_,  
then **SD-WAN** is one of the _tools or implementations_ that helps achieve it.

---

### ✅ **In summary**

- **BWAN** = broader architecture / design philosophy
    
- **SD-WAN** = specific software-defined technology that can implement BWAN principles
    

So you could say:

> “Modern Borderless WAN architectures are typically _powered by SD-WAN_ solutions.”

Would you like a visual diagram comparing their architecture layers?

##### You said:

so if I currently have an SD WAN Fabric, can I integrate BWAN into that?

###### ChatGPT said:

Great, now you’re asking the _right kind_ of architectural question — and the answer is:

> ✅ **Yes**, you _can_ integrate Borderless WAN (BWAN) principles into your existing **SD-WAN fabric**, but it’s not about _adding_ a new physical component — it’s about _extending your current SD-WAN_ with **borderless capabilities** (security, identity, cloud, mobility, and unified policy).

Let’s break down what that really means:

---

## 🧠 1. SD-WAN vs BWAN in Practice

If you already have an **SD-WAN fabric**, that gives you:

- Centralized policy and control
    
- Application-aware routing
    
- Dynamic path selection (MPLS, broadband, LTE/5G, etc.)
    
- Basic security and segmentation
    

But a **Borderless WAN (BWAN)** goes further by extending those benefits _beyond_ the WAN fabric — into the cloud, users, and devices, with unified identity, security, and access control.

---

## ⚙️ 2. How to Integrate BWAN Capabilities into an SD-WAN Fabric

Here’s how you’d “evolve” your SD-WAN into a Borderless WAN architecture:

|BWAN Capability|How to Add/Integrate It|
|---|---|
|**Zero-Trust & Identity-Aware Access**|Integrate **Identity Services Engine (ISE)** or similar for user/device-based policies across WAN edges.|
|**Secure Internet & Cloud Access**|Add **Cloud OnRamp**, **SASE** (Secure Access Service Edge), or **CASB** integrations to extend SD-WAN into cloud edges securely.|
|**Unified Security Policy**|Use **SD-WAN + Secure Firewall** (like Cisco Umbrella or Palo Alto Prisma Access) to enforce consistent policies regardless of location.|
|**Mobility Integration**|Extend WAN policies to mobile/remote users via **VPNaaS** or **ZTP** (Zero-Touch Provisioning).|
|**Automation & Visibility**|Integrate with **Cisco DNA Center**, **vManage**, or **ThousandEyes** for end-to-end visibility, analytics, and automation.|

Essentially, your SD-WAN is the _transport and orchestration layer_, while BWAN principles guide how you unify access, identity, and security _across_ that transport.

---

## 🧩 3. Architecture Evolution Example

**Current State:**  
`Branch ↔ SD-WAN ↔ Data Center/Cloud`

**Target (BWAN-Enabled) State:**  
`User/Device ↔ Secure Edge (SASE/Cloud) ↔ SD-WAN Fabric ↔ Apps (Cloud, Data Center, SaaS)`

Here, the WAN boundaries disappear — users securely connect from _anywhere_ with consistent policy and experience.

---

## 🧭 4. Summary

- You don’t “add” BWAN _to_ SD-WAN.
    
- You **extend your SD-WAN** implementation to align with **BWAN principles** — security everywhere, unified policy, and seamless connectivity across all edges.
    
- In Cisco terms, it’s like moving from _SD-WAN_ → _SASE + Borderless Network Architecture_.
    

---

Would you like me to outline a **migration roadmap** (step-by-step phases) for evolving an SD-WAN into a Borderless WAN environment?

##### You said:

so is BWAN essentially a kind of wrapper around SD WAN that enables the Fabric to look like a full mesh topology, but have succinct and granular control/security policies at the edges "within" the Fabric?

###### ChatGPT said:

That’s a _very sharp and accurate interpretation_ — and you’re **90% right** in how you’re thinking about it. Let’s refine it just slightly so you can describe it precisely in technical and architectural terms.

---

## 🧭 Short answer

> **Yes — BWAN can be thought of as a “wrapper” or higher-level architectural layer around the SD-WAN fabric**, designed to make the network behave _as if it were a seamless, borderless, full-mesh system_ with unified security, control, and policy enforcement at every edge.

But it’s **not only about topology (mesh)** — it’s equally about **context-aware connectivity**, **identity-based policy**, and **consistent security across all network edges** (branches, cloud, remote users, and data centers).

---

## 🧩 Deep dive

Here’s how BWAN “wraps” around the SD-WAN fabric conceptually:

|Layer|Purpose|Technology Enablers|
|---|---|---|
|**BWAN (Architectural Layer)**|Defines the _borderless_ model — unified identity, policy, and security across any edge.|Zero-Trust, SASE, Identity Services Engine (ISE), Secure Edge Gateways, Cloud Access Security Brokers (CASB), DNS-layer security, etc.|
|**SD-WAN (Control / Data Plane Fabric)**|Provides dynamic connectivity and transport independence between network endpoints (branch, DC, cloud).|vManage/vSmart, BGP-EVPN, IPSec tunnels, QoS, path selection, segmentation.|
|**Underlay (Transport)**|Physical or virtual links — MPLS, broadband, LTE/5G, satellite, etc.|ISP circuits, carrier services.|

So the **BWAN layer** governs _how_ the SD-WAN fabric behaves and interacts with users, devices, and clouds — effectively turning your WAN into a **“logical borderless mesh”** where policy follows the user or application, not the link or site.

---

## 🕸️ Topology perspective

You’re right to call out the _“full-mesh illusion.”_

- SD-WAN natively can create **dynamic partial- or full-mesh** topologies for optimal pathing.
    
- BWAN extends that by making **all edges (including remote/cloud)** appear as peers in a global policy and trust domain — a _borderless_ overlay.
    

So from an operational standpoint:

- Each edge (branch, user, cloud gateway) enforces **consistent, granular policy**.
    
- Control and visibility are centralized.
    
- The “borders” between enterprise, cloud, and Internet dissolve.
    

---

## 🔐 Security & control refinement

BWAN adds:

- **Contextual control:** policies based on _who_ or _what_ is connecting, not just IP/subnet.
    
- **Granular segmentation:** micro-segmentation “inside” the fabric, often leveraging tags, groups, or SGTs (Security Group Tags).
    
- **Unified trust model:** integrates with Zero-Trust and SASE components to secure all edges equally.
    

So while SD-WAN gives you the _pipes_ and _orchestration_,  
**BWAN defines the rules of the road** — security, access, and identity awareness — across those pipes.

---

## 🧠 TL;DR

> BWAN ≈ _Architectural “wrapper” around SD-WAN_ that makes the WAN behave like a global, secure, full-mesh overlay — where **policy, security, and identity** are uniformly enforced at every edge.

---

Would you like me to draw (or describe) a **logical diagram** showing how BWAN overlays and interacts with SD-WAN fabric layers? It’s often easier to visualize the wrapper model that way.

##### You said:

so if I already have Cisco SDWAN, it would be a relatively seamless process to integrate that with BWAN?

###### ChatGPT said:

Excellent — that’s exactly the right way to think about it.  
And yes — ✅ if you **already have Cisco SD-WAN**, integrating **Borderless WAN (BWAN)** capabilities is **relatively seamless**, because Cisco _designed_ BWAN as a **natural architectural evolution** of its SD-WAN fabric within the **Cisco Borderless Network** and now **Cisco Secure Access (SASE)** frameworks.

Let’s unpack what “relatively seamless” actually means in practice:

---

## 🧩 1. Why It’s Seamless

Your existing **Cisco SD-WAN (Viptela or IOS-XE SD-WAN)** already provides:

- Dynamic overlay tunnels and full-mesh capabilities
    
- Centralized orchestration via **vManage / vSmart / vBond**
    
- Segmentation and traffic steering
    
- API-driven automation and telemetry
    

These are **core building blocks** for a Borderless WAN.

When you “extend” to BWAN, you’re not replacing this — you’re **enriching** it by integrating:

- **Identity-driven access control** (Cisco ISE, Duo, or similar)
    
- **Unified security and cloud edge** (Cisco Umbrella, SIG, or SASE components)
    
- **Cloud and SaaS integration** (Cloud OnRamp for SaaS/IaaS)
    
- **Analytics and visibility** (ThousandEyes, DNA Center, or SecureX)
    

So instead of re-architecting, you’re **layering BWAN capabilities** on top of your existing SD-WAN fabric.

---

## ⚙️ 2. Integration Building Blocks

Here’s what that typically looks like step by step:

|Integration Area|Cisco Solution|What It Adds|
|---|---|---|
|**Identity & Policy Control**|Cisco ISE + SD-WAN segmentation|Context-aware, identity-driven access within the WAN fabric|
|**Cloud Security & SASE Edge**|Cisco Umbrella SIG or Cisco+ Secure Connect|Extends the SD-WAN fabric to Internet & cloud securely|
|**Zero Trust / Remote Access**|Cisco Duo + AnyConnect (or Secure Client)|Brings users into the BWAN fabric with consistent policies|
|**Cloud OnRamp**|Built into vManage|Direct optimization and policy-based routing to SaaS/IaaS|
|**Visibility & Assurance**|ThousandEyes + vAnalytics + DNA Center|Global visibility across all edges and fabrics|

All of these integrate natively with Cisco SD-WAN using APIs or policy extensions — meaning **no major topology redesigns or forklift upgrades**.

---

## 🔄 3. Migration Path Example

Here’s a practical evolution roadmap:

1. **Baseline:**  
    Fully functional Cisco SD-WAN (branches, DCs, and cloud)
    
2. **Phase 1 – Security Integration:**  
    Integrate Cisco Umbrella SIG & ISE for identity-based policies.
    
3. **Phase 2 – Cloud OnRamp:**  
    Extend SD-WAN fabric directly into Azure, AWS, or Google Cloud.
    
4. **Phase 3 – Borderless Access:**  
    Incorporate remote/mobile users via Cisco+ Secure Connect (SASE).
    
5. **Phase 4 – Unified Analytics:**  
    Add ThousandEyes/DNA Center for end-to-end BWAN visibility.
    

After this, your WAN effectively becomes **“borderless”** — identity-driven, policy-consistent, and cloud-connected.

---

## 🧠 4. Key Takeaway

> If you’re already running Cisco SD-WAN, you’re **60–70% of the way** to a Borderless WAN.  
> The remaining steps are mostly **integration and policy harmonization**, not infrastructure overhaul.

---

Would you like me to outline a **detailed migration checklist** (with prerequisites, integration order, and impact areas) for evolving a Cisco SD-WAN into a Borderless WAN fabric? It’s often what Cisco SEs and network architects use in design workshops.
# New Question
asljf