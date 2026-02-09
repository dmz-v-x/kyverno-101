# Kyverno Policy Types

## Why Policy Types Matter

Kyverno does **four different kinds of work**.

If you don’t separate them mentally:
- Policies become confusing
- Debugging becomes painful
- Interviews become difficult
- Production behavior feels “random”

So we’ll treat each policy type as a **different tool**.

---

## Big Picture First

Kyverno policy types:

- Validate → **Block or warn**
- Mutate → **Modify resources**
- Generate → **Create new resources**
- VerifyImages → **Secure container images**

Each type:
- Runs at a different moment
- Has a different purpose
- Solves a different problem

They are **not interchangeable**.

---

## 1. Validate Policies — Block or Warn

### What Validate Policies Do

Validate policies **check rules**.

They answer questions like:
- Is this allowed?
- Does this follow standards?
- Is something missing or invalid?

They do NOT change anything.

---

## What Validate Can Do

Validate policies can:

- Allow the request
- Reject the request
- Warn (audit) but allow

That’s it.

No fixing. No auto-creation.

---

## Hands-On Thinking: Validate in Real Life

Imagine this rule:

“All Pods must have a team label”

What validate does:
- Looks at the incoming Pod
- Checks if label exists

If missing:
- Reject the Pod  
OR
- Allow but report violation

Validate = **strict checker**

---

## When Validate Policies Run

- CREATE
- UPDATE
- (optionally) DELETE

They run during **validating admission**.

---

## Use Validate When

- You want strict enforcement
- You want visibility without fixing
- You want developers to correct mistakes

Validate policies teach discipline.

---

## 2. Mutate Policies — Modify Resources

### What Mutate Policies Do

Mutate policies **change the request**.

They modify resources **before** they are stored.

They do NOT block by default.

---

## What Mutate Can Do

Mutate policies can:
- Add missing fields
- Change field values
- Inject defaults
- Normalize configurations

Mutation happens automatically.

---

## Hands-On Thinking: Mutate in Real Life

Same rule:

“All Pods must have a team label”

Instead of blocking:
- Kyverno adds the label automatically

Developer applies:
- Minimal YAML

Cluster stores:
- Correct, standardized YAML

Mutate = **auto-fix**

---

## When Mutate Policies Run

- CREATE
- UPDATE

They run during **mutating admission**  
(before validation).

---

## Use Mutate When

- You want developer convenience
- You want platform consistency
- You want fewer deployment failures

Mutate policies reduce friction.

---

## Important Validate vs Mutate Difference

Validate:
- Stops bad behavior

Mutate:
- Fixes bad behavior

In real clusters:
- Both are used together

---

## 3. Generate Policies — Create New Resources

### What Generate Policies Do

Generate policies **create additional resources**.

They do NOT change the original request.

They react to the creation of something else.

---

## What Generate Can Do

Generate policies can:
- Create ConfigMaps
- Create NetworkPolicies
- Create RoleBindings
- Create Secrets (carefully)

Based on another resource being created.

---

## Hands-On Thinking: Generate in Real Life

Example rule:

“When a namespace is created, also create:
- a NetworkPolicy
- a ResourceQuota”

Developer creates:
- Namespace only

Kyverno automatically creates:
- Required supporting resources

Generate = **automation**

---

## When Generate Policies Run

- After the original resource is created
- Can also run in background mode

They are not part of mutation or validation.

---

## Use Generate When

- You want platform defaults
- You want zero manual setup
- You want consistency at scale

Generate policies remove repetitive work.

---

## 4. VerifyImages — Secure Container Images

### What VerifyImages Policies Do

VerifyImages policies:
- Inspect container images
- Enforce image security rules

They are focused **only on images**.

---

## What VerifyImages Can Do

VerifyImages policies can:
- Allow only trusted registries
- Require signed images
- Verify image metadata
- Block untrusted images

This is supply-chain security.

---

## Hands-On Thinking: VerifyImages in Real Life

Example rule:

“Only allow images signed by our organization”

Developer tries:
- Random public image

Result:
- Deployment blocked

Only trusted images pass.

VerifyImages = **security guard for images**

---

## When VerifyImages Policies Run

- During admission
- Before workloads are created

They are a specialized form of validation.

---

## Common Beginner Mistake (Avoid This)

Trying to:
- Validate when mutation is needed
- Mutate when validation is needed
- Generate when mutation would work

Always ask:
“What am I trying to achieve?”

---

## Mental Separation (Lock This In)

Think like this:

- Validate → Judge  
	“Is this allowed?”

- Mutate → Mechanic  
	“I’ll fix this for you”

- Generate → Builder  
	“I’ll create what’s missing”

- VerifyImages → Security  
	“Is this image trustworthy?”

Each has **one job**.

---

## How These Work Together in Real Clusters

Real platforms use:
- Mutate to auto-fix
- Validate to enforce rules
- Generate to bootstrap resources
- VerifyImages to secure supply chain

Kyverno becomes a **policy toolbox**, not a single hammer.
