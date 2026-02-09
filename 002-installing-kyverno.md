# Installing Kyverno & Basic Demo

## Prerequisites (Minimal)

You need:

- A running Kubernetes cluster  
  - minikube OR kind OR any cluster
- kubectl configured and working

Verify first:

	kubectl get nodes

If this works, you’re ready.

---

## Step 1 — What We Are About to Do

We will:

1. Install Kyverno
2. Confirm Kyverno pods are running
3. Confirm Kyverno registered admission webhooks
4. Send a request through Kubernetes
5. Prove Kyverno sits in the request path

At the end, you should **trust** that Kyverno is active.

---

## Step 2 — Install Kyverno (Official Way)

Kyverno is installed using plain Kubernetes YAML.

Run this:

	kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

What just happened?

- A new namespace was created
- Kyverno controllers were deployed
- Admission webhooks were registered

We’ll verify all of that next.

---

## Step 3 — Verify Kyverno Namespace

Check namespaces:

	kubectl get ns

You should see:

- kyverno

This namespace contains everything Kyverno needs.

---

## Step 4 — Inspect Kyverno Pods (Very Important)

Run:

	kubectl get pods -n kyverno

You should see pods like:

- kyverno-admission-controller
- kyverno-background-controller
- kyverno-cleanup-controller
- kyverno-reports-controller

Do NOT rush past this.

Each pod has a purpose:
- Admission controller → intercepts requests
- Background controller → scans existing resources
- Reports controller → generates policy reports

For now, focus on **admission-controller**.

---

## Step 5 — Confirm Kyverno Is an Admission Controller

Admission controllers work via **webhooks**.

List all admission webhooks:

	kubectl get validatingwebhookconfigurations
	kubectl get mutatingwebhookconfigurations

You should see entries starting with:

- kyverno-

This is proof that:
- Kubernetes API Server will call Kyverno
- Every matching request passes through Kyverno

Kyverno is now “wired into” Kubernetes.

---

## Step 6 — Observe Kyverno Admission Logs

Let’s watch Kyverno react to requests.

Open a terminal and run:

	kubectl logs -n kyverno -l app.kubernetes.io/component=admission-controller -f

Leave this running.

This is your **X-ray machine**.

---

## Step 7 — Send a Kubernetes Request (Hands-On Proof)

In another terminal, create a simple pod.

Create file pod.yaml:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: kyverno-demo-pod
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply it:

	kubectl apply -f pod.yaml

You should see:
- Pod created successfully
- Logs appearing in Kyverno terminal

This proves:
- Request passed through Kyverno
- Kyverno inspected the request
- No policy blocked it yet

---

## Step 8 — Why Nothing Happened (Important Insight)

You might think:
“Kyverno didn’t do anything”

That’s expected.

Right now:
- Kyverno is installed
- Kyverno is intercepting requests
- BUT no policies exist

No policy = no action.

This is **good**.
It means Kyverno is passive until you tell it what to enforce.

---

## Step 9 — Mental Model Lock-In

At this point:

- Kubernetes API Server is active
- Kyverno is sitting in front of resource creation
- Every CREATE / UPDATE request flows through Kyverno
- Kyverno waits for rules (policies)

This is the exact foundation needed for:
- Validation policies
- Mutation policies
- Generation policies

---

## Common Beginner Mistakes (Avoid These)

- Thinking Kyverno replaces controllers  
  → It does not

- Expecting Kyverno to act without policies  
  → It won’t

- Not checking webhook configurations  
  → Then debugging becomes guesswork

Always remember:
> If webhook exists, Kyverno is in the request path.
