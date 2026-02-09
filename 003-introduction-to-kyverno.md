# Introduction to Kyverno

## Part 1 — What Is Kyverno?

Let’s start simple.

### Definition (Plain English)

Kyverno is a **policy engine for Kubernetes**.

That means:
- You define **rules (policies)**
- Kubernetes resources must **follow those rules**
- If they don’t:
	- Kyverno can **block**
	- Kyverno can **fix**
	- Kyverno can **warn**

Kyverno sits **inside Kubernetes**, not outside it.

---

## What Does “Policy Engine” Actually Mean?

A policy engine answers questions like:

- Is this resource allowed?
- Is this resource safe?
- Does this resource follow company standards?

Kyverno answers those questions **every time** someone:
- creates a resource
- updates a resource
- (optionally) deletes a resource

---

## Key Thing to Remember

> Kyverno does NOT deploy applications  
> Kyverno controls **how** applications are deployed

---

## Policies in Kyverno (Why Devs Like It)

Kyverno policies are:

- Written as **YAML**
- Look like normal Kubernetes resources
- Stored in Git
- Applied using kubectl
- No new language to learn

That’s a **huge deal**.

You don’t need:
- Rego
- DSLs
- External compilers

If you know Kubernetes YAML → you can learn Kyverno.

---

## Features of Kyverno (Explained Practically)

Let’s convert features into real actions.

---

### 1. Policies as YAML-Based Kubernetes Resources

A Kyverno policy is just another YAML file.

That means:
- kubectl apply works
- GitOps works
- Reviews work
- Rollbacks work

Kyverno feels **native**, not bolted on.

---

### 2. Enforce Policies as an Admission Controller

Kyverno runs when Kubernetes receives requests.

So when you run:

	kubectl apply -f deployment.yaml

Kyverno checks:
- Is this allowed?
- Does it follow rules?

Before Kubernetes stores anything.

---

### 3. Validate, Mutate, Generate, Cleanup Resources

Kyverno can:

- Validate  
	Block bad resources

- Mutate  
	Automatically fix resources

- Generate  
	Create required resources automatically

- Cleanup  
	Remove unwanted resources

All without changing application code.

---

### 4. Verify Container Images (Supply Chain Security)

Kyverno can:
- Allow only approved image registries
- Verify signed images
- Enforce trusted publishers

This protects against:
- Compromised images
- Unknown registries
- Supply chain attacks

---

### 5. Policies Beyond Kubernetes (Advanced)

Kyverno policies can apply to:
- Terraform plans
- Cloud resources
- Authorization payloads

Same policy mindset. Same YAML style.

---

### 6. Policies as Code (GitOps Friendly)

Kyverno policies:
- Live in Git
- Reviewed like application code
- Versioned
- Auditable

This is **policy as code**, done right.

---

## Part 2 — Why Do We Need Kyverno?

Now let’s expose the real problem.

---

## What kubectl apply Actually Does

Imagine this file.

	deployment.yaml

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: nginx
	spec:
	  replicas: 1
	  template:
	    spec:
	      containers:
	        - name: nginx
	          image: nginx

You deploy it:

	kubectl apply -f deployment.yaml

What Kubernetes checks:

- Is YAML syntax valid?
- Is API version correct?
- Does this resource already exist?

That’s it.

---

## What Kubernetes Does NOT Check

Kubernetes does NOT care if:

- You forgot labels
- You forgot resource limits
- You used an unsafe image
- You violated company standards
- You broke security rules

If syntax is valid → Kubernetes accepts it.

This is **dangerous at scale**.

---

## Real-World Cluster Problem

In real organizations:
- Many teams
- Many developers
- Many environments

Without policies:
- Clusters drift
- Security breaks
- Standards disappear
- Debugging becomes painful

Kubernetes gives **freedom**  
Kyverno gives **control**

---

## Part 3 — Types of Policies We Want (Real Examples)

Let’s look at real problems Kyverno solves.

---

### Example 1 — Enforce Labels on Deployments

Problem:
- You don’t know which team owns what

Rule:
- Every Deployment must have a team label

Without Kyverno:
- Developers forget
- No enforcement

With Kyverno:
- Block creation if label is missing
OR
- Automatically add label

---

### Example 2 — Allow Images Only From Approved Registry

Problem:
- Anyone can pull images from anywhere

Risk:
- Malware
- Unknown publishers

Kyverno rule:
- Only allow images from approved registries

If violated:
- Block deployment

---

### Example 3 — Enforce Unique Ingress Hostnames

Problem:
- Two teams use same hostname
- Traffic breaks

Kyverno rule:
- Reject duplicate ingress hostnames

---

### Example 4 — Require Resource Limits

Problem:
- Pods consume unlimited CPU/memory
- Node crashes

Kyverno rule:
- Every container must define resource limits

---

### Example 5 — Deny Traffic Without Network Policies

Problem:
- Pods can talk to everything
- Security nightmare

Kyverno rule:
- Require network policy before allowing workloads

---

## What Kyverno Can Do With Violations

Kyverno does NOT always block.

It can:

- Block  
	Stop the request

- Mutate  
	Fix the resource automatically

- Warn  
	Allow but report violation

This flexibility is why platform teams love Kyverno.

---

## Mental Model to Lock In

Think like this:

- Kubernetes → accepts anything valid
- Kyverno → enforces standards
- Policies → rules of the cluster
- Developers → free but guarded
