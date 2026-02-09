# Kyverno Policy Structure

## Big Picture First — What Is a Kyverno Policy?

A Kyverno policy is just a **Kubernetes resource**.

That means:
- It has apiVersion
- It has kind
- It has metadata
- It has spec

Inside `spec`, Kyverno-specific logic lives.

---

## Minimal Skeleton of Any Kyverno Policy

Every Kyverno policy follows this shape:

	apiVersion: kyverno.io/v1
	kind: Policy OR ClusterPolicy
	metadata:
	  name: policy-name
	spec:
	  rules:
	    - name: rule-name
	      match:
	        ...
	      validate OR mutate OR generate OR verifyImages:
	        ...

Everything else is built on top of this.

---

## 1. apiVersion

	apiVersion: kyverno.io/v1

What it means:
- This tells Kubernetes which API group owns this resource
- Kyverno registers this API group when installed

Important:
- If apiVersion is wrong → policy is rejected
- This is not optional

You almost never change this.

---

## 2. kind — Policy vs ClusterPolicy (VERY IMPORTANT)

This decides **scope**.

### Policy

	kind: Policy

Scope:
- Namespace-scoped
- Applies only inside one namespace

Use when:
- Different namespaces need different rules
- Teams manage their own policies

Example use case:
- Dev namespace rules
- Team-specific enforcement

---

### ClusterPolicy

	kind: ClusterPolicy

Scope:
- Cluster-wide
- Applies to ALL namespaces

Use when:
- Organization-wide standards
- Security rules
- Platform-level enforcement

Rule of thumb:
> If it must apply everywhere → ClusterPolicy

---

## Example — Policy vs ClusterPolicy

Namespace policy:

	kind: Policy
	metadata:
	  name: require-team-label
	  namespace: dev

Cluster-wide policy:

	kind: ClusterPolicy
	metadata:
	  name: require-team-label

---

## 3. metadata

	metadata:
	  name: require-team-label

What metadata does:
- Identifies the policy
- Used for auditing and reporting

Rules:
- Name must be unique
- Use descriptive names
- Avoid generic names like policy1

Metadata does NOT control behavior.

---

## 4. spec — Where Kyverno Logic Starts

	spec:
	  rules:
	    - ...

The spec field:
- Contains all Kyverno-specific configuration
- Controls how policies behave

Everything important lives under `spec`.

---

## 5. rules — The Heart of a Policy

	spec:
	  rules:
	    - name: rule-1
	    - name: rule-2

Rules are:
- Evaluated independently
- Applied sequentially
- Each rule has one responsibility

Best practice:
- One rule = one idea

---

## 6. rule.name

	name: require-team-label

Why this matters:
- Appears in logs
- Appears in policy reports
- Helps debugging

Never skip meaningful rule names.

---

## 7. match — Which Resources This Rule Applies To

	match:
	  resources:
	    kinds:
	      - Pod

Match answers:
> “Which requests should this rule apply to?”

You can match by:
- Kind
- Namespace
- Name
- Labels
- Operations (CREATE, UPDATE)

---

### Match Example — Only Pods

	match:
	  resources:
	    kinds:
	      - Pod

Only Pod requests are evaluated.

---

### Match Example — Only Deployments in prod

	match:
	  resources:
	    kinds:
	      - Deployment
	    namespaces:
	      - prod

---

## 8. exclude — Which Resources to Ignore

	exclude:
	  resources:
	    namespaces:
	      - kube-system

Exclude is the opposite of match.

Use exclude to:
- Protect system namespaces
- Avoid breaking core components

---

### Exclude Example — Ignore kube-system

	exclude:
	  resources:
	    namespaces:
	      - kube-system

This is extremely common in real clusters.

---

## 9. validate — Block or Warn (Complete Example)

### Goal:
All Pods must have a team label.

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
	        message: "team label is required"
	        pattern:
	          metadata:
	            labels:
	              team: "?*"

What happens:
- Pod without label → blocked
- Pod with label → allowed

Validate never modifies anything.

---

## 10. mutate — Modify Resources (Complete Example)

### Goal:
Automatically add team label if missing.

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

What happens:
- Pod comes in
- Kyverno injects label
- Modified Pod is stored

Mutate happens BEFORE validation.

---

## 11. generate — Create New Resources (Complete Example)

### Goal:
Create a NetworkPolicy when a namespace is created.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: generate-network-policy
	spec:
	  rules:
	    - name: generate-np
	      match:
	        resources:
	          kinds:
	            - Namespace
	      generate:
	        kind: NetworkPolicy
	        name: default-deny
	        namespace: "{{request.object.metadata.name}}"
	        synchronize: true
	        data:
	          spec:
	            podSelector: {}
	            policyTypes:
	              - Ingress
	              - Egress

What happens:
- Namespace created
- Kyverno creates NetworkPolicy
- synchronize keeps it in sync

Generate does NOT change the original request.

---

## 12. verifyImages — Secure Container Images (Complete Example)

### Goal:
Allow images only from a trusted registry.

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: restrict-image-registry
	spec:
	  rules:
	    - name: verify-registry
	      match:
	        resources:
	          kinds:
	            - Pod
	      verifyImages:
	        - imageReferences:
	            - "myregistry.io/*"
	          required: true

What happens:
- Pod using untrusted image → blocked
- Trusted image → allowed

verifyImages is image-specific security.

---

## How All Fields Fit Together (Mental Model)

Think in layers:

- apiVersion → who owns this resource
- kind → scope (namespace vs cluster)
- metadata → identity
- spec → behavior
- rules → individual checks
- match/exclude → where rule applies
- validate/mutate/generate/verifyImages → what action to take

Each layer has ONE job.

---

## Common Beginner Mistakes (Avoid These)

- Using Policy instead of ClusterPolicy accidentally
- Forgetting exclude for kube-system
- Mixing validate and mutate in same rule
- Writing giant rules instead of small ones
- Not naming rules properly
