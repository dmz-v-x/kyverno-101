# Validating & Mutating Webhooks vs Kyverno

## 1. Kubernetes Admission Control (Foundation)

Kubernetes does not immediately accept changes to cluster state.

Whenever you try to:
- create a resource
- update a resource
- delete a resource

Kubernetes sends that request through **Admission Control**.

Admission control exists to:
- protect the cluster
- enforce rules
- ensure correctness

This happens **before** the object is stored in ETCD.

---

## 2. Types of Admission Webhooks in Kubernetes

Kubernetes supports **two webhook types** as part of admission control:

1. Mutating Webhooks  
2. Validating Webhooks  

They serve **different purposes** and run in a **fixed order**.

---

## 3. Mutating Webhooks (First Phase)

### What Is a Mutating Webhook?

A **Mutating Webhook** can:
- inspect a resource
- modify the resource
- add defaults
- inject fields

It runs **before validation**.

---

### What Mutating Webhooks Are Used For

Common use cases:
- Add labels or annotations
- Inject sidecars
- Set default values
- Modify image names
- Add securityContext

Important rule:
> A mutating webhook can change the resource, but must return a valid object.

---

### What Mutating Webhooks Cannot Do

- They cannot skip schema validation
- They cannot persist resources themselves
- They should not contain complex business logic

---

## 4. Validating Webhooks (Second Phase)

### What Is a Validating Webhook?

A **Validating Webhook**:
- inspects a resource
- decides allow or deny
- does NOT modify the resource

It runs **after mutation and schema validation**.

---

### What Validating Webhooks Are Used For

Common use cases:
- Enforce required labels
- Block privileged containers
- Enforce image rules
- Enforce naming conventions
- Enforce security policies

Validating webhooks are **gatekeepers**.

---

## 5. Full Kubernetes Admission Workflow (Correct Order)

The complete request flow is:

API Request  
→ API Server  
→ Authentication  
→ Authorization  
→ Mutating Admission  
→ Schema Validation  
→ Validating Admission  
→ Store in ETCD  

This order is **fixed**.

---

## 6. Power of Native Kubernetes Webhooks

Validating and Mutating Webhooks are:
- extremely powerful
- very flexible
- low-level primitives

You can enforce **almost anything** using them.

So the obvious question arises…

---

## 7. Why Do We Need Kyverno If Webhooks Already Exist?

The answer is NOT:
- “Webhooks are bad”
- “Kyverno replaces webhooks”

The real answer is:
> Webhooks are **low-level**, Kyverno is **high-level**

---

## 8. The Problem With Native Kubernetes Webhooks

Kubernetes gives you the *mechanism*, not the *solution*.

If you want to use webhooks directly, **you must do everything yourself**.

---

## 9. What You Must Do With Native Webhooks

To enforce policies using native webhooks, you must:

1. Write a webhook server  
	- Build an HTTP service  
	- Handle AdmissionReview requests  

2. Implement policy logic  
	- Parse Kubernetes objects  
	- Check labels, fields, values  

3. Return admission responses  
	- Allow or deny  
	- Handle mutations correctly  

4. Deploy and maintain the server  
	- Secure with TLS  
	- Scale and monitor  

5. Register webhook configurations  
	- ValidatingWebhookConfiguration  
	- MutatingWebhookConfiguration  

This approach is powerful — but **heavy and error-prone**.

---

## 10. Example: Native Validating Webhook (Mental Model)

To enforce:
“All Deployments must have an env label”

You must:
- Write custom code
- Inspect metadata.labels
- Handle edge cases
- Maintain logic over time

Every new rule means:
- More code
- More testing
- More maintenance

---

## 11. Enter Kyverno (Why It Exists)

Kyverno exists to solve **this exact pain**.

Kyverno is:
- A Kubernetes-native policy engine
- Built on top of admission webhooks
- Designed to eliminate custom webhook code

Kyverno **uses webhooks internally**.

---

## 12. Core Idea Behind Kyverno

With Kyverno:
- You write **policies**
- Kyverno handles **webhooks**
- You do NOT write webhook servers
- You do NOT write custom code

You define *what* you want.  
Kyverno figures out *how* to enforce it.

---

## 13. Declarative Policy Language (Major Difference)

### Native Webhooks
- Imperative
- Code-driven
- Hard to audit

### Kyverno
- Declarative
- YAML-based
- Git-friendly
- Human-readable

Example Kyverno policy:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: require-env-label
	spec:
	  rules:
	    - name: check-env
	      match:
	        resources:
	          kinds:
	            - Deployment
	      validate:
	        message: "env label is required"
	        pattern:
	          metadata:
	            labels:
	              env: "?*"

No server. No code. No webhook config.

---

## 14. Kubernetes-Native by Design

Kyverno is designed **for Kubernetes**, not adapted later.

That means:
- Policies are Kubernetes resources
- Policies live in ETCD
- Policies are versioned
- Policies are auditable
- Policies work with kubectl, Git, Kustomize

---

## 15. Policy Management at Scale

Native webhooks:
- Policies scattered in code
- Hard to audit and change

Kyverno:
- Centralized policy management
- Easy enable/disable
- Easy rollback
- Safe updates

---

## 16. Policy Reuse and Templates

Native webhooks:
- Reuse is manual
- Copy-paste common

Kyverno:
- Reusable patterns
- Consistent syntax
- Lower error rate

---

## 17. Advanced Features Kyverno Adds

Kyverno provides features beyond basic webhooks:

1. Policy Reports  
	- Track violations  
	- Audit compliance  

2. Advanced Mutation  
	- Conditional mutation  
	- Defaulting logic  

3. Generate Policies  
	- Auto-create resources  
	- Enforce platform defaults  

---

## 18. Native Webhook Workflow (End-to-End)

Traditional workflow:

1. Write webhook server  
2. Deploy service  
3. Register webhook  
4. Kubernetes sends requests  
5. Server evaluates logic  
6. Kubernetes enforces decision  

Flexible — but complex.

---

## 19. Kyverno Workflow (End-to-End)

Kyverno workflow:

1. Install Kyverno  
2. Kyverno registers webhooks automatically  
3. Define policies (Policy / ClusterPolicy)  
4. Kubernetes sends requests to Kyverno  
5. Kyverno evaluates policies  
6. Kubernetes enforces decision  

You never touch webhook code.

---

## 20. Who Manages Webhooks?

Native approach:
- You manage everything

Kyverno:
- Kyverno manages:
	- webhook registration
	- certificates
	- updates
	- failure handling

---

## 21. Failure Handling & Safety

Kyverno integrates safely with Kubernetes:

- Secure by default
- Handles timeouts
- Rotates certificates automatically

Manually doing this is hard.

---

## 22. When to Use Native Webhooks

Use native webhooks when:
- You need extremely custom behavior
- You want full control
- You can maintain custom code

---

## 23. When to Use Kyverno

Use Kyverno when:
- You want declarative policies
- You want no custom code
- You want auditing and reporting
- You want scalable policy management

---

## 24. Kyverno’s Role in Kubernetes

Kyverno is:
- NOT a replacement for webhooks
- NOT a replacement for Kubernetes
- NOT a competitor to admission control

Kyverno is:
> A higher-level abstraction built on top of admission webhooks

---

## 25. Mental Model to Lock In

Think like this:

- Kubernetes → provides hooks  
- Webhooks → low-level plumbing  
- Kyverno → policy engine using that plumbing  
- Policies → human-defined rules  

Kyverno makes **hard things easy**.

---

## 26. Final Summary

- Kubernetes supports validating and mutating webhooks
- Webhooks are powerful but low-level
- Native webhook usage requires custom servers
- Kyverno abstracts webhook complexity
- You define policies, Kyverno enforces them
- Kyverno is Kubernetes-native, declarative, scalable


