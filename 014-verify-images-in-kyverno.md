# `verifyImages` in Kyverno 

## 1. Why Image Security Matters (Real Problem)

In Kubernetes, this is allowed by default:

	spec:
	  containers:
	    - image: randomuser/nginx:latest

Kubernetes does NOT check:
- Who built the image
- Where it came from
- Whether it was tampered with
- Whether it’s trusted

If syntax is valid → Kubernetes accepts it.

This is a **supply-chain security risk**.

---

## 2. What verifyImages Is (Plain English)

verifyImages is a **Kyverno policy type** that:

- Inspects container images
- Enforces trust rules
- Blocks untrusted images
- Protects the supply chain

verifyImages works during **admission**, before workloads run.

---

## 3. Where verifyImages Runs in the Request Flow

Request flow reminder:

API Request  
→ API Server  
→ Authentication  
→ Authorization  
→ Mutating Admission  
→ Schema Validation  
→ **Validating Admission (verifyImages runs here)**  
→ Store in ETCD  

verifyImages behaves like **specialized validation**, focused only on images.

---

## 4. What verifyImages Can Enforce

verifyImages can enforce:

- Approved image registries
- Image name patterns
- Signed images
- Image metadata constraints
- Required signatures (Cosign)

In this blog, we start with **registry enforcement**, which is the most common.

---

## 5. verifyImages Policy Skeleton (Memorize This)

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: image-policy
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: image-rule
	      match:
	        resources:
	          kinds:
	            - Pod
	      verifyImages:
	        - imageReferences:
	            - "myregistry.io/*"
	          required: true

This is the basic structure.

---

## 6. Example 1 — Allow Images Only From a Trusted Registry

### Goal  
Only allow images from `myregistry.io`.

---

### Step 1 — Create the Policy

Create `allow-only-trusted-registry.yaml`:

	apiVersion: kyverno.io/v1
	kind: ClusterPolicy
	metadata:
	  name: allow-only-trusted-registry
	spec:
	  validationFailureAction: Enforce
	  rules:
	    - name: check-image-registry
	      match:
	        resources:
	          kinds:
	            - Pod
	      verifyImages:
	        - imageReferences:
	            - "myregistry.io/*"
	          required: true

Apply it:

	kubectl apply -f allow-only-trusted-registry.yaml

---

### Step 2 — Try an Untrusted Image

Create a Pod:

	apiVersion: v1
	kind: Pod
	metadata:
	  name: bad-image
	spec:
	  containers:
	    - name: nginx
	      image: nginx

Apply it:

	kubectl apply -f bad-image.yaml

---

### Step 3 — Observe the Failure

You will see an error like:

	admission webhook denied the request:
	image verification failed

This happens **before** the Pod is created.

---

### Step 4 — Try a Trusted Image

	apiVersion: v1
	kind: Pod
	metadata:
	  name: good-image
	spec:
	  containers:
	    - name: nginx
	      image: myregistry.io/nginx:1.0

This is allowed.

---

## 7. imageReferences (Important Detail)

	imageReferences:
	  - "myregistry.io/*"

This supports:
- Wildcards
- Repository-level rules
- Tag patterns

Examples:
- myregistry.io/*  
- myregistry.io/backend/*  
- ghcr.io/org/*  

---

## 8. verifyImages Applies to All Containers

verifyImages checks:
- containers
- initContainers
- ephemeral containers

If ANY container violates the rule:
- The request is blocked

This is intentional.

---

## 9. Using Audit Mode First (Best Practice)

Never enforce image rules immediately.

Start with:

	validationFailureAction: Audit

What happens:
- Pods are allowed
- Violations are recorded
- No outages

Check reports:

	kubectl get policyreports -A

Once clean → switch to Enforce.

---

## 10. Excluding System Namespaces (CRITICAL)

Always exclude system namespaces:

	exclude:
	  resources:
	    namespaces:
	      - kube-system
	      - kyverno

System components often use images you don’t control.

---

## 11. verifyImages vs validate (Important Difference)

validate:
- Checks YAML structure
- Checks values inside spec

verifyImages:
- Talks to registries
- Evaluates image trust
- Focused on supply chain

They complement each other.

---

## 12. Common verifyImages Mistakes (Avoid These)

- Enforcing too early
- Forgetting audit mode
- Blocking system namespaces
- Allowing public registries accidentally
- Assuming developers understand image trust

Image policies must be **communicated clearly**.

---

## 13. Real-World verifyImages Rollout Strategy

Safe rollout:

1. Start with Audit
2. Observe violations
3. Educate teams
4. Fix image sources
5. Enforce gradually
6. Monitor reports

Never jump directly to Enforce.

---

## 14. Mental Model to Lock In

Think of verifyImages as:

- A border security checkpoint
- Every container image must show ID
- Untrusted images are turned away

This is **supply-chain security**, not YAML validation.
