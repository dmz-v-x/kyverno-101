# Advanced Validate Policies in Kyverno

## 1. Why Basic Validate Rules Are Not Enough

So far, you validated things like:
- required labels
- required limits

That’s great — but real clusters are messy.

Real problems:
- Multiple acceptable configurations
- Optional fields
- Different teams, different patterns
- Controllers vs Pods
- Partial specs

This is where **advanced validation** matters.

---

## 2. pattern vs anyPattern (CRITICAL DIFFERENCE)

### pattern
- Enforces **exact structure**
- All specified fields must match
- Too strict if used blindly

### anyPattern
- Allows **multiple valid shapes**
- At least **one pattern must match**
- Safer for production

Rule of thumb:
> Use pattern for strict rules  
> Use anyPattern for flexibility

---

## 3. Example 1 — pattern (Strict Validation)

### Goal  
All Pods must define BOTH cpu and memory limits.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: strict-resource-limits
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: require-cpu-memory
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "cpu and memory limits are mandatory"
	        pattern:
	          spec:
	            containers:
	              - resources:
	                  limits:
	                    cpu: "?*"
	                    memory: "?*"

Behavior:
- Missing either cpu or memory → BLOCK
- Very strict

Use when:
- You control all workloads
- Platform-owned clusters

---

## 4. Problem With Strict pattern

What if:
- Init containers exist?
- Multiple containers exist?
- Sidecars are injected?
- One container is exempt?

Strict pattern can **accidentally block valid workloads**.

Solution → anyPattern.

---

## 5. Example 2 — anyPattern (Flexible Validation)

### Goal  
Allow Pods if:
- All containers have limits  
OR
- Pod belongs to a trusted namespace

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: flexible-resource-limits
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: limits-or-trusted
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "Pods must have limits unless in trusted namespace"
	        anyPattern:
	          - spec:
	              containers:
	                - resources:
	                    limits:
	                      cpu: "?*"
	                      memory: "?*"
	          - metadata:
	              namespace: trusted

Behavior:
- Either pattern passing is enough
- Much safer rollout

---

## 6. Validating Nested Objects (Very Common)

Kubernetes specs are deeply nested.

### Example — Block Privileged Containers

### Goal  
Containers must NOT run as privileged.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: block-privileged
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: no-privileged
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "privileged containers are not allowed"
	        pattern:
	          spec:
	            containers:
	              - securityContext:
	                  privileged: false

Important detail:
- If privileged is true → BLOCK
- If privileged is missing → FAILS

This might be too strict.

---

## 7. Handling Optional Fields (Gotcha)

If a field might be missing, strict pattern breaks workloads.

### Safer Version (anyPattern)

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: no-privileged-safe
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: no-privileged
	      match:
	        resources:
	          kinds:
	            - Pod
	      validate:
	        message: "privileged containers are not allowed"
	        anyPattern:
	          - spec:
	              containers:
	                - securityContext:
	                    privileged: false
	          - spec:
	              containers:
	                - securityContext: {}

Meaning:
- privileged=false → OK
- securityContext missing → OK
- privileged=true → BLOCK

This is **production-safe validation**.

---

## 8. Validating Multiple Containers Correctly

Common mistake:
- Validating only first container

Kyverno patterns apply to **all array items**.

So this works correctly:

	spec:
	  containers:
	    - resources:
	        limits:
	          cpu: "?*"

Kyverno checks **each container**, not just one.

---

## 9. Validating on UPDATE Requests (Important)

By default:
- validate applies to CREATE and UPDATE

That means:
- Scaling
- Image updates
- Config changes

All are validated.

Be careful:
- Too strict rules can block updates
- Especially on legacy workloads

---

## 10. Safe Rollout Strategy (Senior Tip)

Never do this first:

	validationFailureAction: Enforce

Instead:
1. Start with Audit
2. Observe PolicyReports
3. Fix violations
4. Switch to Enforce

This avoids production incidents.

---

## 11. Excluding System Namespaces (Always Do This)

Almost every validate policy should include:

	exclude:
	  resources:
	    namespaces:
	      - kube-system
	      - kyverno

Otherwise:
- You WILL break system components

This is non-negotiable.

---

## 12. Debugging Validate Failures Like a Pro

When something is blocked:

1. Read the error message
2. Check rule name
3. Check policy name
4. Inspect PolicyReport
5. Inspect resource YAML

Validate failures are deterministic — no guessing needed.

---

## 13. Common Validate Mistakes (Checklist)

Avoid these:
- Overusing strict pattern
- Forgetting anyPattern
- Blocking system namespaces
- Enforcing too early
- Mixing mutate logic into validate
- Writing giant rules

One rule = one responsibility.

---

## 14. Mental Model to Lock In

Think like this:

- pattern → exact shape
- anyPattern → allowed shapes
- validate → judge, not fixer
- audit → observe
- enforce → block

This mindset prevents outages.
