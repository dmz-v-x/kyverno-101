# What Are Kubernetes Admission Controllers?

## What Happens When You Create or Change Something in Kubernetes?

When you try to:
- deploy an application
- update a deployment
- delete a resource

Kubernetes does **not** accept your request immediately.

Instead, your request goes through a **checkpoint system** called  
**Admission Control**.

Think of admission control as **security guards at the gate**.

---

## What Is Admission Control?

Admission Control is a phase inside the Kubernetes API Server where:

- Requests are **inspected**
- Requests can be:
	- Allowed
	- Modified
	- Rejected

This happens **before** the object is stored in the cluster.

Important:
> Admission control runs for **every request** that wants to  
> CREATE, UPDATE, or DELETE a Kubernetes resource.

This is true even if:
- You are not using Kyverno
- You did not install any policy engine

---

## Admission Control Is Always Active

This is a very common misunderstanding, so lock this in:

> Kubernetes **always** uses admission control.

Even a brand-new Kubernetes cluster has admission controllers enabled.

They are part of Kubernetes itself.

---

## Where Admission Control Happens (Request Flow)

When you run:

	kubectl apply -f deployment.yaml

The flow is:

1. kubectl sends request
2. API Server receives request
3. Authentication happens
4. Authorization happens
5. **Admission Controllers run**
6. Object is stored in etcd

Admission controllers run **after auth**, but **before storage**.

---

## What Admission Controllers Can Do

Admission controllers can:

- Allow the request  
	“Looks good, proceed”

- Modify the request  
	“I’ll adjust this before saving”

- Reject the request  
	“This is not allowed”

This decision happens **once per request**.

---

## Types of Admission Controllers in Kubernetes

Kubernetes has **two categories** of admission controllers:

1. Built-in Admission Controllers  
2. Dynamic Admission Controllers  

They serve different purposes.

---

## Built-In Admission Controllers

Built-in admission controllers are:

- Part of Kubernetes
- Available by default
- Run in every cluster (unless explicitly disabled)

They enforce **basic safety and correctness**.

You don’t install them manually.

---

### Examples of Built-In Admission Controllers

#### NamespaceLifecycle

Purpose:
- Protect critical namespaces

What it does:
- Prevents deletion of system namespaces
- Ensures namespace lifecycle rules are respected

Without this:
- You could accidentally delete core namespaces

---

#### LimitRanger

Purpose:
- Enforce resource limits

What it does:
- Applies default CPU/memory limits
- Enforces min/max constraints

Without this:
- Pods could consume unlimited resources

---

#### ServiceAccount

Purpose:
- Manage service accounts

What it does:
- Automatically assigns a service account to pods
- Injects credentials where required

Without this:
- Pods may fail to authenticate properly

---

## Key Point About Built-In Controllers

Built-in admission controllers:
- Are **generic**
- Solve **basic cluster needs**
- Are **not customizable** for organization-specific rules

This is where they fall short.

---

## Dynamic Admission Controllers

Dynamic admission controllers are:

- Optional
- Installed by users
- Used for **custom policies**

Examples:
- Kyverno
- OPA Gatekeeper

Kubernetes only sends requests to them **if you install them**.

---

## What Makes Dynamic Admission Controllers Powerful?

They allow you to define rules like:

- Enforce labels
- Restrict image registries
- Enforce security standards
- Enforce naming conventions
- Enforce organizational policies

These rules are **not built into Kubernetes**.

---

## How Kubernetes Uses Dynamic Admission Controllers

When a request reaches the admission phase:

- Kubernetes checks built-in controllers
- Then checks dynamic admission controllers (if installed)

If Kyverno is installed:
- Kubernetes forwards the request to Kyverno
- Kyverno evaluates your policies
- Kyverno responds with:
	- allow
	- modify
	- reject

---

## Important Clarification

Dynamic admission controllers do **not replace** built-in ones.

They **extend** Kubernetes.

Built-in controllers:
- Handle core safety

Dynamic controllers:
- Handle organization-specific rules

Both run together.

---

## Mental Model to Lock In

Think like this:

- Built-in admission controllers  
	→ default security guards hired by Kubernetes

- Dynamic admission controllers  
	→ custom security guards hired by your organization

Kyverno is a **custom guard**, not a replacement.

---

## Why This Matters for Kyverno

Kyverno works because:

- Kubernetes supports dynamic admission controllers
- Kubernetes sends requests to Kyverno automatically
- Kyverno responds with policy decisions

If admission control didn’t exist:
- Kyverno could not work at all
