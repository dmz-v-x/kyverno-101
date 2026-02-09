# Hands-On Validate Policies in Kyverno

## 1. What a Validate Policy Actually Does (Quick Reset)

A **validate policy**:
- Checks a resource
- Does **NOT** modify it
- Can:
	- Block the request
	- OR allow it with a warning (audit)

Think of validate as:
> “Show me what you’re trying to create. I’ll decide if it’s allowed.”

---

## 2. Cluster Setup Assumption

Before continuing, make sure:

- Kyverno is installed
- Cluster is running
- kubectl works

Quick check:

	kubectl get pods -n kyverno

If Kyverno pods are running → you’re good.

---

## 3. Validate Policy Skeleton (Memorize This)

Every validate policy follows this structure:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: policy-name
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: rule-name
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "error message"
	        pattern:
	          ...

You’ll reuse this **every time**.

---

## 4. Example 1 — Require a Label on Pods (Basic & Critical)

### Goal
Every Pod must have a `team` label.

---

### Step 1 — Create the Policy

Create a file `require-team-label.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: require-team-label
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: check-team-label
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "team label is required on all Pods"
	        pattern:
	          metadata:
	            labels:
	              team: "?*"

Apply it:

	kubectl apply -f require-team-label.yaml

---

### Step 2 — Try to Create a Pod WITHOUT the Label

Create `bad-pod.yaml`:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: bad-pod
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply it:

	kubectl apply -f bad-pod.yaml

---

### Step 3 — Observe the Failure

You should see:

	Error from server (Forbidden):
	admission webhook "validate.kyverno.svc" denied the request:
	team label is required on all Pods

🎯 **This is Kyverno blocking a workload in real time.**

---

## 5. Fix the Pod (Developer Experience)

Now fix the Pod:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: good-pod
	  labels:
	    team: platform
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply again:

	kubectl apply -f good-pod.yaml

Pod is created successfully.

---

## 6. Key Learning from Example 1

You learned:

- validate.pattern enforces structure
- "?*" means:
	- field must exist
	- value must not be empty
- Kyverno blocks **before ETCD**
- Error messages come from `validate.message`

This is the **core validate behavior**.

---

## 7. Example 2 — Require Resource Limits (Very Common)

### Goal
All containers must define CPU and memory limits.

---

### Step 1 — Create the Policy

Create `require-resource-limits.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: require-resource-limits
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: check-limits
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "CPU and memory limits are required"
	        pattern:
	          spec:
	            containers:
	              - resources:
	                  limits:
	                    cpu: "?*"
	                    memory: "?*"

Apply it:

	kubectl apply -f require-resource-limits.yaml

---

### Step 2 — Try a Pod Without Limits

	apiVersion: v1
	kind: Pod
	metadata:
	  name: no-limits
	  labels:
	    team: platform
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply it:

	kubectl apply -f no-limits.yaml

Result:
- Request is rejected
- Clear error message returned

---

### Step 3 — Fix the Pod

	apiVersion: v1
	kind: Pod
	metadata:
	  name: with-limits
	  labels:
	    team: platform
	spec:
	  containers:
	    - name: nginx
	      image: nginx
	      resources:
	        limits:
	          cpu: "500m"
	          memory: "256Mi"

Now it works.

---

## 8. Example 3 — Validate on Deployments (Not Just Pods)

Validate policies apply to **any resource**.

### Goal
All Deployments must have an `env` label.

---

Create `require-env-label.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: require-env-label
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: check-env-label
	      match:
	        resources:
	          kinds:
	            - Deployment
	      validate:
	        message: "env label is required on Deployments"
	        pattern:
	          metadata:
	            labels:
	              env: "?*"

Try deploying a Deployment without the label → blocked.

---

## 9. Enforce vs Audit Mode (Very Important)

So far we used:

	validationFailureAction: Enforce

This means:
- Violations = BLOCK

Now change it to:

	validationFailureAction: Audit

What happens:
- Resource is allowed
- Violation is recorded
- No blocking

This is how teams **gradually roll out policies**.

---

## 10. Where Do Violations Appear?

Run:

	kubectl get policyreports -A

You’ll see:
- Which resources violated which rules
- Which policies were triggered

This is how platform teams audit clusters safely.

---

## 11. Common Beginner Mistakes (Avoid These)

- Forgetting to exclude kube-system
- Writing too many checks in one rule
- Using validate when mutate is needed
- Not providing clear error messages
- Enforcing too early without audit phase

---

## 12. Mental Model to Lock In

Validate policies are:

- Strict
- Predictable
- Non-invasive
- Best for:
	- Security
	- Compliance
	- Guardrails

Validate teaches **discipline**.

