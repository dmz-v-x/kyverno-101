# Mutate Policies in Kyverno

## 1. What a Mutate Policy Really Does

A **mutate policy**:
- Modifies the incoming resource
- Runs during **mutating admission**
- Happens **before validation**
- Changes are invisible to the user unless they inspect the resource

Think of mutate as:
> “I’ll fix this for you so it meets standards.”

---

## 2. When You SHOULD Use Mutate (Very Important)

Use mutate when:
- The fix is obvious
- The value can be defaulted safely
- You don’t want to break deployments
- You want consistency without friction

Do NOT use mutate when:
- A human decision is required
- Security rules must be explicit
- Silent fixing would hide mistakes

---

## 3. Mutate Policy Skeleton (Memorize This)

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: mutate-policy-name
	spec:
	  rules:
	    - name: rule-name
	      match:
	        resources:
	          kinds:
	            - Pod
	      mutate:
	        patchStrategicMerge:
	          ...

Everything lives inside `patchStrategicMerge`.

---

## 4. Example 1 — Auto-Add a Missing Label (Classic)

### Goal  
If a Pod does not have a `team` label, add it automatically.

---

Create `mutate-add-team-label.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: add-team-label
	spec:
	  rules:
	    - name: mutate-team-label
	      match:
	        resources:
	          kinds:
	            - Pod
	      mutate:
	        patchStrategicMerge:
	          metadata:
	            labels:
	              team: platform

Apply it:

	kubectl apply -f mutate-add-team-label.yaml

---

### Test It

Create a Pod **without** the label:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: demo-pod
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply it.

Now inspect the Pod:

	kubectl get pod demo-pod -o yaml

You’ll see:

	labels:
	  team: platform

Kyverno mutated it **before storage**.

---

## 5. patchStrategicMerge — What’s Really Happening

patchStrategicMerge:
- Merges fields into existing YAML
- Does NOT overwrite unrelated fields
- Is Kubernetes-aware (arrays, maps)

This is why mutation is safe.

---

## 6. Example 2 — Add Default Resource Limits

### Goal  
If containers don’t define limits, add defaults.

---

Create `mutate-default-limits.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: default-resource-limits
	spec:
	  rules:
	    - name: add-default-limits
	      match:
	        resources:
	          kinds:
	            - Pod
	      mutate:
	        patchStrategicMerge:
	          spec:
	            containers:
	              - resources:
	                  limits:
	                    cpu: "500m"
	                    memory: "256Mi"

Apply it.

---

### Test Without Limits

Create a Pod without limits.

After creation:

	kubectl get pod <pod-name> -o yaml

Limits are now present.

This is a **huge developer-experience win**.

---

## 7. Example 3 — Mutate Based on Namespace (Very Common)

### Goal  
Add `env=prod` label only in prod namespace.

---

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: add-env-label-prod
	spec:
	  rules:
	    - name: prod-env-label
	      match:
	        resources:
	          kinds:
	            - Pod
	          namespaces:
	            - prod
	      mutate:
	        patchStrategicMerge:
	          metadata:
	            labels:
	              env: prod

This mutation applies **only in prod namespace**.

---

## 8. Important Mutate Gotcha (Silent Changes)

Mutation is silent:
- kubectl apply succeeds
- User may not realize changes happened

Best practice:
- Document mutations
- Pair mutate with validate
- Avoid surprising behavior

---

## 9. Combining Mutate + Validate (PRODUCTION PATTERN)

This is extremely important.

### Strategy:
1. Mutate to fix easy things
2. Validate to block if still wrong

---

### Example — Labels Policy (Best Practice)

Policy 1 — Mutate missing label:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: mutate-team-label
	spec:
	  rules:
	    - name: add-team
	      match:
	        resources:
	          kinds:
	            - Pod
	      mutate:
	        patchStrategicMerge:
	          metadata:
	            labels:
	              team: platform

Policy 2 — Validate label exists:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: validate-team-label
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: check-team
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "team label is required"
	        pattern:
	          metadata:
	            labels:
	              team: "?*"

Mutation runs first → validation passes.

This is **gold-standard Kyverno design**.

---

## 10. Example 4 — Mutate Deployment Templates (Very Common)

Mutate applies to:
- Pod
- Deployment
- StatefulSet
- Job

For Deployments, mutate the Pod template:

	spec:
	  template:
	    metadata:
	      labels:
	        team: platform

Example:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: mutate-deployment-label
	spec:
	  rules:
	    - name: add-label
	      match:
	        resources:
	          kinds:
	            - Deployment
	      mutate:
	        patchStrategicMerge:
	          spec:
	            template:
	              metadata:
	                labels:
	                  team: platform

---

## 11. Common Mutate Mistakes (Avoid These)

- Mutating security-sensitive fields silently
- Forgetting namespace scoping
- Overwriting instead of merging
- Mutating system namespaces
- Using mutate when validate is better

---

## 12. Always Exclude System Namespaces

Add this almost always:

	exclude:
	  resources:
	    namespaces:
	      - kube-system
	      - kyverno

This avoids breaking cluster internals.

---

## 13. Mental Model to Lock In

Think like this:

- Mutate → auto-fix
- Validate → enforce
- Mutate first, validate second
- Defaults first, discipline later

This keeps clusters safe AND friendly.


