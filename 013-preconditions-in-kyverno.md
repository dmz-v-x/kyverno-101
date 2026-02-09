# Preconditions in Kyverno

## 1. Why Preconditions Exist (Very Important)

So far, your policies look like this:

- Match resource
- Validate or mutate
- Enforce rules

But here’s the problem:

> Kyverno policies apply to **every matching request**  
> even when it **should not**

Examples of bad behavior without preconditions:
- Blocking UPDATE when only CREATE should be checked
- Mutating already-correct resources
- Blocking system-managed changes
- Breaking legacy workloads

Preconditions solve this.

---

## 2. What Is a Precondition?

A **precondition** is a condition that must be true  
**before** a rule is evaluated.

Think of it like:

> “Only apply this rule IF these conditions are met”

If the precondition fails:
- The rule is skipped
- No validation
- No mutation
- No blocking

This is **critical for safety**.

---

## 3. Where Preconditions Live in a Policy

Preconditions are defined **inside a rule**.

Structure:

	spec:
	  rules:
	    - name: rule-name
	      match:
	        ...
	      preconditions:
	        ...
	      validate OR mutate:
	        ...

They are evaluated:
- After match
- Before validate / mutate

---

## 4. Two Types of Preconditions

Kyverno provides **two logical modes**:

1. preconditions.all  
2. preconditions.any  

This is equivalent to:
- AND logic
- OR logic

---

## 5. preconditions.all (AND Logic)

### Meaning

All conditions must be true  
for the rule to execute.

If **any one fails**, rule is skipped.

---

### Example 1 — Apply Policy Only on CREATE

### Goal  
Validate Pods **only when they are created**, not updated.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: validate-on-create-only
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: require-team-label-on-create
	      match:
	        resources:
	          kinds:
	            - Pod
	      preconditions:
	        all:
	          - key: "{{request.operation}}"
	            operator: Equals
	            value: CREATE
	      validate:
	        message: "team label is required on pod creation"
	        pattern:
	          metadata:
	            labels:
	              team: "?*"

Behavior:
- CREATE → validated
- UPDATE → skipped
- DELETE → skipped

This avoids blocking updates later.

---

## 6. Why This Matters (Real Production Insight)

Without this:
- Editing a Pod label could be blocked
- Scaling Deployments could fail
- Rolling updates could break

Preconditions protect **day-2 operations**.

---

## 7. preconditions.any (OR Logic)

### Meaning

If **any one condition is true**,  
the rule executes.

Used when **multiple acceptable triggers** exist.

---

### Example 2 — Apply Policy on CREATE or UPDATE

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: validate-on-create-or-update
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: enforce-label
	      match:
	        resources:
	          kinds:
	            - Pod
	      preconditions:
	        any:
	          - key: "{{request.operation}}"
	            operator: Equals
	            value: CREATE
	          - key: "{{request.operation}}"
	            operator: Equals
	            value: UPDATE
	      validate:
	        message: "team label is required"
	        pattern:
	          metadata:
	            labels:
	              team: "?*"

---

## 8. request.operation (Critical Variable)

One of the most important context variables:

	request.operation

Possible values:
- CREATE
- UPDATE
- DELETE

This allows **surgical control** over policy execution.

Most production policies should use this.

---

## 9. Preconditions with Mutate (Very Common Pattern)

### Problem  
Mutation keeps running again and again on UPDATE.

### Solution  
Run mutation **only if something is missing**.

---

### Example 3 — Mutate Only If Label Is Missing

### Goal  
Add team label **only if it doesn’t already exist**.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: mutate-team-label-once
	spec:
	  rules:
	    - name: add-team-label
	      match:
	        resources:
	          kinds:
	            - Pod
	      preconditions:
	        all:
	          - key: "{{request.object.metadata.labels.team}}"
	            operator: Equals
	            value: null
	      mutate:
	        patchStrategicMerge:
	          metadata:
	            labels:
	              team: platform

Behavior:
- Missing label → mutation runs
- Label exists → mutation skipped

This avoids unnecessary mutations.

---

## 10. Using request.object (Powerful but Dangerous)

	request.object

This gives you access to:
- The entire incoming resource
- All fields and values

Examples:
- metadata.labels
- spec.containers
- spec.securityContext

This is powerful — but misuse causes bugs.

---

## 11. Preconditions on Nested Fields

### Example 4 — Mutate Only If Limits Are Missing

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: default-limits-only-if-missing
	spec:
	  rules:
	    - name: add-default-limits
	      match:
	        resources:
	          kinds:
	            - Pod
	      preconditions:
	        all:
	          - key: "{{request.object.spec.containers[].resources.limits}}"
	            operator: Equals
	            value: null
	      mutate:
	        patchStrategicMerge:
	          spec:
	            containers:
	              - resources:
	                  limits:
	                    cpu: "500m"
	                    memory: "256Mi"

Meaning:
- If limits missing → add defaults
- If limits exist → do nothing

This prevents overwriting user intent.

---

## 12. Common Preconditions Mistakes (Avoid These)

- Forgetting preconditions on UPDATE
- Overusing Equals without checking null
- Applying rules to DELETE accidentally
- Writing conditions that always evaluate true
- Using mutate without guard conditions

Every mutation should ask:
> “Should this really run every time?”

---

## 13. Preconditions + Validate (Best Practice)

Typical production pattern:

1. Mutate if missing (with preconditions)
2. Validate strictly (with preconditions)
3. Enforce only on CREATE
4. Audit on UPDATE

This balances:
- Safety
- Flexibility
- Stability

---

## 14. Mental Model to Lock In

Think of preconditions as:

- Safety switches
- Execution guards
- Context filters

Match answers:
> “Which resources?”

Preconditions answer:
> “Under what conditions?”

Both are required for safe policies.
