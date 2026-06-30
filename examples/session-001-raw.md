# Knowledge Extraction: Kubernetes networking troubleshooting
**Session:** session-001
**Expert:** Alex Chen, Platform Engineer
**Date:** 2025-06-30
**Purpose:** rag
**Depth:** practitioner

---

<!-- Interview transcript below — appended during /knowledge-interview -->

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
