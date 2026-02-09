# Kyverno Architecture

## Why Kyverno Architecture Matters

Many people can *write* Kyverno policies.  
Very few understand **how Kyverno actually works under the hood**.

If you understand the architecture:
- Debugging becomes easy
- Performance issues make sense
- Interview answers become confident
- Production behavior is predictable

So let’s break Kyverno down properly.

---

## High-Level Idea: How Kyverno Works

Kyverno works by **plugging itself into Kubernetes admission control**.

It does NOT:
- Replace Kubernetes
- Bypass the API server
- Act after resources are stored

Kyverno works **before** resources are saved.

---

## Kyverno Core Components (Internal Architecture)

Kyverno is made up of **multiple controllers**, each with a focused job.

Think of Kyverno as a **team**, not a single process.

---

## 1. Webhook

The **Webhook** is Kyverno’s entry point.

What it does:
- Listens for requests from Kubernetes
- Receives admission review requests
- Sends responses back to the API server

Important:
- Kubernetes talks to Kyverno **only through webhooks**
- Without webhooks, Kyverno cannot intercept requests

Webhook = Kyverno’s ears.

---

## 2. Engine (Policy Engine)

The **Engine** is the brain of Kyverno.

What it does:
- Evaluates policies
- Matches rules against resources
- Decides whether to:
	- Allow
	- Mutate
	- Deny
	- Audit

The engine does NOT talk to Kubernetes directly.

It only:
- Receives requests from the webhook
- Applies logic
- Returns decisions

Engine = decision maker.

---

## 3. Webhook Controller

Policies change over time.

New policies:
- Are added
- Updated
- Deleted

The **Webhook Controller** ensures that:
- Webhooks are always aware of current policies
- Admission rules stay in sync
- Kubernetes knows when to call Kyverno

In simple terms:
- It keeps Kyverno’s webhook configuration **up to date**

Webhook Controller = configuration manager.

---

## 4. Cert Renewer

Kyverno uses **TLS certificates** for secure communication.

Problem:
- Certificates expire

Solution:
- Cert Renewer automatically renews certificates
- Prevents webhook failures
- Keeps communication secure

Without this:
- Kyverno would suddenly stop working
- Requests could fail unexpectedly

Cert Renewer = security maintenance.

---

## 5. Background Controller

Admission controllers only handle **new requests**.

But what about:
- Resources that already exist?
- Old deployments created before Kyverno?

That’s where the **Background Controller** comes in.

What it does:
- Scans existing resources
- Applies policies in background mode
- Detects violations without blocking workloads

This is crucial for:
- Gradual adoption
- Auditing
- Compliance checks

Background Controller = retroactive enforcement.

---

## 6. Report Controller

Policies are useless if you can’t see results.

The **Report Controller**:
- Generates PolicyReport objects
- Tracks pass/fail results
- Updates policy violation status

These reports are used for:
- Auditing
- Monitoring
- CI/CD feedback
- Compliance visibility

Report Controller = visibility & reporting.

---

## Summary of Kyverno Components

Think of Kyverno like this:

- Webhook → receives requests
- Engine → evaluates policies
- Webhook Controller → keeps webhook synced
- Cert Renewer → maintains security
- Background Controller → scans existing resources
- Report Controller → shows results

Each component does **one job well**.

---

## General Kubernetes Request Flow (Baseline)

Before adding Kyverno, Kubernetes already follows this flow:

API Request  
→ API Server  
→ Authentication  
→ Authorization  
→ Admission Control  
→ Store in ETCD  

Admission Control is where **all policy engines live**.

---

## Complete Admission Workflow (Detailed)

Let’s zoom into the admission phase.

Full request path:

API Request  
→ API Server  
→ Authentication / Authorization  
→ Mutating Admission  
→ Schema Validation  
→ Validating Admission  
→ Store in ETCD  

This sequence is **fixed** by Kubernetes.

---

## Mutating Admission (First Phase)

Purpose:
- Modify the request

What happens here:
- Default values may be added
- Fields may be injected
- Configurations may be adjusted

Kyverno can:
- Add labels
- Inject security settings
- Modify container specs

Important:
> Mutation happens **before validation**

---

## Schema Validation (Middle Phase)

Kubernetes checks:
- API version correctness
- Required fields
- Object structure

This is Kubernetes-native validation.

No custom logic happens here.

---

## Validating Admission (Final Phase)

Purpose:
- Approve or reject the request

What happens here:
- Policies are evaluated
- Decisions are final

Kyverno can:
- Reject non-compliant resources
- Warn instead of blocking
- Enforce strict rules

---

## Where Kyverno Fits in This Workflow

Kyverno integrates via **webhooks** in:

- Mutating Admission
- Validating Admission

Kubernetes:
- Sends the request to Kyverno
- Waits for Kyverno’s response
- Acts based on that response

Kyverno never bypasses Kubernetes rules.

---

## Why Webhooks Are the Key

Webhooks allow Kubernetes to:
- Call external systems
- Delegate decisions
- Extend behavior safely

Kyverno uses webhooks so that:
- Policy logic lives outside Kubernetes core
- Kubernetes remains stable
- Custom enforcement becomes possible

This is **exactly** how Kyverno “does its magic”.

---

## Mental Model to Lock In

Think of the workflow like this:

- Kubernetes → traffic controller
- Admission phases → checkpoints
- Webhooks → phone calls to policy engines
- Kyverno → expert advisor answering the call

Kubernetes asks:
“Is this request okay?”

Kyverno replies:
“Yes, no, or let me fix it first.”
