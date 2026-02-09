# How Kyverno Works

## High-Level Idea (Before Details)

Kyverno works by **standing in the request path** of Kubernetes.

Whenever someone tries to:
- create something
- update something

Kubernetes pauses and asks Kyverno:

> “Is this request okay according to the rules?”

Kyverno then decides what should happen next.

---

## Step 1 — Kyverno Acts as a Dynamic Admission Controller

Kyverno is a **dynamic admission controller**.

This means:
- It is NOT built into Kubernetes
- You install it yourself
- Kubernetes talks to it using **webhooks**

Once installed, Kubernetes knows:
- “For certain requests, I must ask Kyverno first”

---

## What Does “Listening Through Webhooks” Mean?

A **webhook** is simply a callback.

In this case:
- Kubernetes API Server sends a request to Kyverno
- Kyverno processes the request
- Kyverno sends back a response

This happens synchronously.

Kubernetes waits for Kyverno’s answer before proceeding.

---

## What Triggers Kyverno?

Kyverno is triggered when:

- A resource is **created**
- A resource is **updated**
- (optionally) a resource is deleted

Example actions:
- kubectl apply
- kubectl edit
- Helm install
- CI/CD deployment

Each of these sends a request through the API Server.

---

## Step 2 — Kubernetes Sends the Request to Kyverno

When a request reaches the admission phase, Kubernetes sends Kyverno a message that says:

- What resource is being created or updated
- The full YAML object
- Metadata like:
	- Namespace
	- Operation type (CREATE / UPDATE)
	- User info

Conceptually, Kubernetes is asking:

> “Someone wants to add or change this object.  
> Do your policies allow this?”

---

## Step 3 — Kyverno Loads All Relevant Policies

Kyverno already has access to:
- All policies installed in the cluster
- Cluster-wide policies
- Namespace-specific policies

Policies are stored as Kubernetes resources, so:
- Kyverno watches them continuously
- No restart is needed when policies change

At this stage:
- Kyverno knows **what rules exist**

---

## Step 4 — Kyverno Matches the Request Against Policies

Not all policies apply to all resources.

Kyverno first checks:
- Does this policy apply to this request?

It can match resources based on:

- Resource type  
	Pod, Deployment, Service, Ingress, etc

- Resource name  
	Specific objects or patterns

- Namespace  
	Apply only in certain namespaces

- Labels  
	Match based on labels on the resource

- Operation type  
	CREATE vs UPDATE

If a policy does NOT match:
- Kyverno skips it

If it DOES match:
- Kyverno evaluates it further

---

## Step 5 — Kyverno Evaluates the Rules

For each matching policy, Kyverno checks the rules defined by you.

Rules can say things like:

- All Pods must have a specific label
- Containers must not run as root
- Resource limits must be defined
- Certain fields must not exist
- If something is missing, add it

Kyverno compares:
- The incoming resource
- Against the rule conditions

This is where policy logic lives.

---

## Step 6 — Kyverno Decides What to Do

Based on the policy type and rule result, Kyverno can decide to:

- Allow the request  
	Everything is compliant

- Modify the request  
	Add or change fields before saving

- Reject the request  
	Block it completely

- Warn / audit  
	Allow but record a violation

This decision is returned to Kubernetes.

---

## Step 7 — Kubernetes Acts on Kyverno’s Response

Kubernetes receives Kyverno’s response and:

- If allowed  
	→ continues processing the request

- If mutated  
	→ stores the modified object

- If rejected  
	→ returns an error to the user

From the user’s perspective:
- It looks like Kubernetes made the decision
- Kyverno stays invisible but powerful

---

## Important Clarification

Kyverno:
- Does NOT directly create resources
- Does NOT store anything in ETCD
- Does NOT bypass Kubernetes logic

Kyverno only:
- Advises Kubernetes during admission
- Enforces rules you define

Kubernetes always stays in control.

---

## Why This Design Is Powerful

This design means:

- Policies are enforced centrally
- Developers don’t need to change code
- CI/CD pipelines don’t need custom checks
- Standards are enforced consistently

Every request is treated the same way.

---

## Mental Model to Lock In

Think of Kyverno like this:

- Kubernetes → final authority
- Admission phase → decision point
- Kyverno → rule evaluator
- Policies → company rules

Kubernetes asks:
“Is this allowed?”

Kyverno answers:
“Yes, no, or let me fix it.”
