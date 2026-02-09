Kubernetes Controllers & Admission Controllers

## Part 1 — What Is a Controller?

### The Core Problem Kubernetes Solves

Imagine this situation:

- You say:  
  “I want **3 pods** running”
- One pod crashes
- Kubernetes must **automatically fix it**

Who does that fixing?

👉 **Controllers**

---

## Definition — Controller (Simple Words)

A **controller** is a program that:

- Watches the Kubernetes cluster
- Compares:
  - what you *want* (desired state)
  - what *exists* (actual state)
- Takes action to **make reality match your desire**

---

## Desired State vs Actual State

Kubernetes works on a loop:

1. You declare something (YAML)
2. Kubernetes stores it
3. Controllers continuously check:
   - “Is the cluster matching this?”

If not → controller acts.

This is called a **control loop**.

---

## Example — Deployment Controller (Hands-On Thinking)

You apply this:

	apiVersion: apps/v1
	kind: Deployment
	spec:
	  replicas: 3

What you declared:
- Desired state → 3 Pods

What if:
- Only 2 pods are running?

Controller reaction:
- Creates 1 more pod

If:
- 4 pods exist?

Controller reaction:
- Deletes 1 pod

📌 Controllers **continuously reconcile state**

---

## Important Truth

> Kubernetes itself does **nothing automatically**  
> Controllers make Kubernetes “alive”

No controller = no automation.

---

## Common Kubernetes Controllers

You don’t install these — they already exist:

- Deployment controller
- ReplicaSet controller
- Job controller
- Node controller
- Endpoint controller

Each controller:
- Watches specific resources
- Has a **single responsibility**

---

## Part 2 — What Is an Admission Controller?

Now comes the **critical concept for Kyverno**.

Controllers act **after** resources exist.

But what if you want to:
- Block bad resources?
- Modify resources *before* creation?

That’s where **Admission Controllers** come in.

---

## Definition — Admission Controller

An **Admission Controller** is a gatekeeper that:

- Intercepts requests sent to the Kubernetes API Server
- Runs **before** data is saved
- Can:
  - Allow the request
  - Reject the request
  - Modify the request

📌 It works **before controllers ever see the object**

---

## Where Admission Controllers Sit (Flow)

When you run:

	kubectl apply -f pod.yaml

The flow is:

1. kubectl sends request
2. API Server receives request
3. Authentication happens
4. Authorization happens
5. Admission Controllers run
6. Object is stored in etcd
7. Controllers start working

📌 Kyverno lives at step 5

---

## Two Types of Admission Controllers

### 1. Mutating Admission Controller

Purpose:
- Modify the request

Runs:
- First

Examples:
- Add default labels
- Inject sidecars
- Add securityContext
- Modify image names

---

### 2. Validating Admission Controller

Purpose:
- Approve or reject request

Runs:
- After mutation

Examples:
- Block privileged containers
- Require labels
- Enforce image registries
- Enforce resource limits

---

## Key Difference: Controller vs Admission Controller

| Controller | Admission Controller |
|-----------|---------------------|
| Runs after object exists | Runs before object is saved |
| Fixes state | Guards entry |
| Reconciles continuously | Executes once per request |
| Cannot block creation | Can block creation |

📌 Kyverno is **not** a normal controller.

Kyverno is an **Admission Controller**.

---

## Part 3 — Create / Update / Delete Requests (Very Important)

Admission Controllers don’t run randomly.  
They run based on **request type**.

Let’s break them down.

---

## CREATE Request

Triggered when:
- New resource is created

Examples:
- kubectl apply (new object)
- kubectl create
- Helm install

Admission behavior:
- Mutation allowed
- Validation allowed
- Most Kyverno policies target CREATE

---

## UPDATE Request

Triggered when:
- Existing resource is modified

Examples:
- kubectl apply (existing object)
- kubectl edit
- Scaling replicas
- Updating image versions

Admission behavior:
- Mutation allowed
- Validation allowed
- Policies must consider:
  - old object
  - new object

📌 Many production bugs happen due to ignoring UPDATE logic.

---

## DELETE Request

Triggered when:
- Resource is deleted

Examples:
- kubectl delete pod
- Helm uninstall

Important:
- DELETE requests usually:
  - Cannot be mutated
  - Rarely validated

Use cases:
- Prevent deleting critical resources
- Protect namespaces
- Protect production workloads

📌 Kyverno **can** validate delete operations, but carefully.

---

## Why Request Types Matter for Kyverno

Kyverno policies often say:

- Apply only on CREATE
- Apply on CREATE + UPDATE
- Ignore DELETE

If you don’t understand request types:
- Policies may behave unexpectedly
- Production outages can occur

---

## Mental Model to Lock In

Think like this:

- Controllers → janitors (cleaning & fixing later)
- Admission Controllers → security guards (checking entry)
- CREATE → new person entering
- UPDATE → person changing clothes
- DELETE → person leaving


