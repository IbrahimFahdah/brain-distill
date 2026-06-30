# Annotated Transcript: Kubernetes networking troubleshooting
**Session:** session-001
**Reviewed at:** 2025-06-30T11:00:00Z
**Reviewer notes:** Transcript is accurate. Added one missing topic (CNI plugin behaviour) and confirmed the payment retry edge case is the top priority item.

## Priority knowledge (human-identified)

1. **NetworkPolicy deny-by-default on first policy** — most engineers don't know this until they've broken production.
2. **Retry logic on payment endpoints** — caused a real double-billing incident; now enforced by a linter.
3. **Always use fully qualified service names cross-namespace** — silently fails otherwise; hours of debugging time lost.

---

## Original transcript

**Q:** Walk me through how you think about debugging a pod that can't reach another service inside the cluster.

**A:** The first thing I always check is whether the DNS name resolves. Most people go straight to curl or netcat, but if the name doesn't resolve you're debugging the wrong layer. Run `kubectl exec` into the pod and do `nslookup <service-name>.<namespace>.svc.cluster.local`. If that fails, the problem is CoreDNS or the service itself doesn't exist yet.

---

**Q:** What breaks when CoreDNS is the culprit? How do you tell it's CoreDNS and not just a missing service?

**A:** If `nslookup` returns NXDOMAIN, the service probably doesn't exist or is in the wrong namespace. If it hangs or returns SERVFAIL, that's CoreDNS itself having a problem — check its pods are running and look at its logs. The gotcha is that CoreDNS pods sometimes get scheduled on nodes that are themselves having network issues, so the symptom looks like DNS is broken cluster-wide but it's really just the nodes that CoreDNS landed on.

---

**Q:** You mentioned namespace — is that a common mistake, getting the namespace wrong?

**A:** Constantly. The service exists, DNS name is right, but the namespace suffix is wrong. The safe pattern is to always use the fully qualified name: `<service>.<namespace>.svc.cluster.local`. Inside the same namespace you can omit the namespace and it resolves fine, but the moment you cross namespaces without the full name, it silently fails. I've seen engineers spend two hours on this.

---

**Q:** After DNS resolves, what's your next step if the connection still fails?

**A:** Check NetworkPolicies. If DNS resolves but TCP won't connect, NetworkPolicy is the number one cause in our cluster. The tricky part is that NetworkPolicies are additive and deny-by-default once any policy exists in a namespace — so adding your first NetworkPolicy to a namespace silently blocks all traffic that wasn't explicitly allowed, including traffic that was working before.

---

**Q:** What's the failure mode that's burned you personally — something you still remember?

**A:** The retry rule with payment endpoints. We had a service that used exponential backoff with retries on all outbound calls. Someone copy-pasted the HTTP client config into the payment service without reading it. A network timeout hit a payment endpoint, the retry fired, and the customer got charged twice because the first request had actually succeeded — the response just got lost. We had to add idempotency key enforcement after that. Now there's a linter rule that flags any retry decorator on a class in the payments package.

---

## Reviewer additions and corrections

### Addition: CNI plugin behaviour on node restart (gap filled by reviewer)

**Q (reviewer):** Is there anything about node-level networking that affects pod connectivity that we didn't cover?

**A (reviewer):** Yes — when a node restarts, the CNI plugin re-initialises pod network interfaces. Pods that were running before the restart may briefly lose connectivity to other pods on the same node even after the kubelet marks them Ready, because the CNI hasn't fully restored the veth pairs. This is most visible with Flannel and Calico in VXLAN mode. The symptom is pod-to-pod failures that resolve on their own within 30–60 seconds after a node comes back. Don't restart pods to fix it — wait.

---

### Correction: CoreDNS scheduling clarification

The original answer said "nodes that CoreDNS landed on" — to be precise, the issue is that CoreDNS pods use `hostNetwork: false` and rely on the node's network stack. If a node's kube-proxy rules are stale (common after a control-plane upgrade), DNS queries routed to CoreDNS pods on that node will fail, while other nodes are fine. Check kube-proxy pod logs on the affected node if DNS failures appear node-specific.

---

## Gaps acknowledged as out of scope

- Service mesh (Istio/Linkerd) networking — separate domain, not part of vanilla Kubernetes troubleshooting
- Ingress controller debugging — deliberately excluded; Alex's team uses a managed ingress, not self-managed
