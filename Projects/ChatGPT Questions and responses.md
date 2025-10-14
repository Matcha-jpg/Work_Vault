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

# New Question
