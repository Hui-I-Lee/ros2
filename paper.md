# Draft Paper

## Abstract

This paper investigates the feasibility of using **ROS 2 communication over Cilium-managed Kubernetes clusters** with multicast enabled. Although Cilium provides experimental multicast support, our evaluation across eight deployment scenarios demonstrates a discrepancy: communication succeeds only in cases where unicast discovery fallback is triggered, while multicast traffic is consistently dropped. Furthermore, in scenarios without multicast, packets frequently experience loops leading to *TTL exceeded* drops, resulting in complete communication failure. These findings highlight fundamental limitations of Cilium’s multicast implementation for real-time robotics middleware and provide guidelines for deployment choices.

---

## 1. Introduction

ROS 2 relies on the DDS (Data Distribution Service) standard, which by default uses IP multicast for discovery and unicast for data transmission. This discovery mechanism poses challenges when deploying ROS 2 in cloud-native environments where networking layers such as CNI (Container Network Interface) and overlay tunnels may not fully support multicast.

Cilium, a popular eBPF-based CNI, claims experimental multicast support. However, its interaction with `kubeProxyReplacement=true` and different deployment modes (direct-routing vs. overlay) has not been systematically studied in the context of ROS 2. This work evaluates Cilium’s multicast behavior under eight controlled scenarios, distinguishing between successful and failed communication cases.

---

## 2. Methodology

### 2.1 Deployment Environment

* **Kubernetes distribution**: K3s
* **CNI plugin**: Cilium, installed with:

  ```bash
  cilium install \
    --set kubeProxyReplacement=true \
    --set hubble.enabled=true \
    --set prometheus.enabled=true
  ```
* **Configuration**: `multicast-enabled: "true"` in `cilium-config`.
* **Routing mode**: Direct-routing (no overlay tunnel).

### 2.2 Experimental Design

We tested **eight scenarios**, varying:

1. Node placement (same node vs. different node).
2. Service exposure (with vs. without load balancer).
3. Discovery method (with vs. without multicast).

Publisher and subscriber pods communicated using ROS 2 DDS default settings, while logs were collected from Cilium/Hubble to analyze packet handling.

---

## 3. Results

### 3.1 Successful Communication (Scenarios 1–4)

* Logs showed repeated:

  ```
  Multicast handled DROPPED (UDP)
  ```
* However, subsequent unicast transmissions were **FORWARDED**:

  ```
  to-overlay FORWARDED (UDP)
  to-endpoint FORWARDED (UDP)
  ```
* **Interpretation**: Multicast packets were dropped by Cilium, but ROS 2’s unicast discovery fallback established connections. Hence, communication succeeded despite multicast loss.

### 3.2 Failed Communication (Scenarios 5–8)

* Logs contained:

  ```
  TTL exceeded DROPPED (UDP)
  ```
* No successful `to-endpoint FORWARDED` events were observed.
* **Interpretation**: Packets entered routing loops within Cilium’s BPF datapath, causing TTL expiration. Without multicast discovery or unicast fallback, ROS 2 communication failed completely.

---

## 4. Discussion

1. **Multicast Enabled but Ineffective**

   * Even with `multicast-enabled: "true"`, Cilium drops multicast packets, reflecting incomplete L2 multicast flooding support.
   * Thus, ROS 2 relies heavily on unicast fallback for discovery.

2. **Impact of `kubeProxyReplacement=true`**

   * The BPF-based service handling introduces NAT/forwarding rules that may conflict with multicast.
   * In scenarios without multicast, these rules result in TTL-exceeded loops, preventing ROS 2 discovery.

3. **Node Local vs. Cross-Node**

   * Successful cases occur both intra-node and inter-node, as long as unicast fallback is reachable.
   * Failures are correlated with missing unicast fallback, not strictly with topology.

---

## 5. Conclusion

This study demonstrates that:

* **Scenarios 1–4** (with multicast enabled) succeed, but only due to **unicast fallback**, not true multicast support.
* **Scenarios 5–8** fail due to TTL expiration loops when multicast is disabled or inaccessible.
* Cilium’s multicast support remains insufficient for ROS 2 discovery, especially in direct-routing mode with kube-proxy replacement.

**Implication**: Deployments requiring ROS 2 over Kubernetes with Cilium should either:

* Configure DDS vendors (e.g., CycloneDDS, FastDDS) to use unicast-only discovery, or
* Use CNIs with full multicast support when multicast discovery is essential.

