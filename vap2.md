Here’s the documentation-first design—written as if it were the internal spec your team can circulate, review, and implement. It emphasizes the why, what, and how more than templates, and it keeps the implementation choices we converged on:
	•	Enforce every policy cluster-wide by default (single binding per policy).
	•	Namespace exemptions for Platform are embedded inside each policy (literal allowlist in CEL). No labels or annotations.
	•	Business exceptions are per-object, time-boxed entries in one cluster-scoped param (ExceptionRegistry).
	•	Clear RBAC, CI, audit, and operational guidance.

⸻

VAP Exception Framework — Design & Operations Guide

0) Executive Summary

This framework standardizes admission control across clusters using Kubernetes ValidatingAdmissionPolicy (VAP). It establishes cluster-wide enforcement by default, limits namespace-wide exemptions to a Platform-controlled allowlist embedded in each policy, and offers business-friendly per-object exceptions governed by tickets and expiry.

Key outcomes
	•	Strong, consistent guardrails without webhooks.
	•	Auditable, time-boxed exceptions with minimal YAML for app teams.
	•	No namespace metadata toggles (no labels/annotations); exemptions live in policy code (Platform-owned).

⸻

1) Scope & Non-Goals

In-scope
	•	Policies for native and custom resources (Pods, KafkaTopic CRDs, ConfluentRoleBinding CRDs, etc.).
	•	Policy-embedded namespace exemptions (Platform only).
	•	Per-object exceptions for business workloads via one param object.
	•	GitOps flow with CI validation and auditability.

Out-of-scope
	•	Mutation (use MutatingAdmissionPolicy or webhooks if needed).
	•	Automatic exception creation or auto-renewal.
	•	Runtime policy analytics (can be added later via audit logs / SIEM).

⸻

2) Assumptions & Prereqs
	•	Kubernetes ≥ 1.26 with admissionregistration.k8s.io/v1 (we run RKE2 1.31.x).
	•	Flux or ArgoCD (GitOps) applies manifests from a Platform-owned repo.
	•	App teams submit exceptions via a separate repo; they do not apply resources directly.
	•	Time synchronized across controllers and CI runners (for expiry logic in CI).

⸻

3) Definitions & Concepts
	•	VAP: ValidatingAdmissionPolicy—CEL-based admission checks (native).
	•	VAPB: ValidatingAdmissionPolicyBinding—applies a policy cluster-wide and supplies paramRef.
	•	ExceptionRegistry: cluster-scoped param CR (single object) that contains only per-object exceptions (no namespace exemptions).
	•	policyId: stable string key used in policies (e.g., noPrivileged) and referenced by exception entries.
	•	Per-object exception: a small record that applies narrowly (namespace + labels or names, optional kinds) and expires.

⸻

4) Roles & Responsibilities

Role	Responsibilities
Platform Engineering	Author and own VAPs; embed namespace allowlists; manage bindings; operate GitOps + CI; review exception PRs; incident response.
Business/App Teams	Propose per-object exceptions via vap-exceptions repo; include ticket, reason, expiry; keep scope narrow; close/renew exceptions.
Security/Governance	Define acceptable rule sets; review high-impact changes; audit exception volumes and durations.


⸻

5) Architecture Overview

                          (Platform repo)                 (Business repo)
                        ┌───────────────────┐           ┌──────────────────┐
                        │   vap-constraints │           │  vap-exceptions  │
                        │  (applied by GitOps)          │   (not applied)  │
                        ├────────┬──────────┤           ├────────┬─────────┤
                        │  VAPs  │ Bindings │           │  PRs   │  CI     │
                        └───┬────┴────┬─────┘           └────────┴─────────┘
                            │         │
                            │         │ paramRef → ExceptionRegistry (applied)
                            │         ▼
                        ┌───────────────────┐   consults exceptions at admission time
                        │ ExceptionRegistry │◄───────────────────────────────┐
                        └─────────┬────────┘                                  │
                                  │                                           │
        ┌─────────────────────────┴────────────────────────┐                  │
        │             Kubernetes API Server                │                  │
        │   ValidatingAdmissionPolicy evaluator (CEL)      │                  │
        └──────────────────────────────────────────────────┘                  │
                    ▲                                      ▲                 │
      PER-NAMESPACE └─ ns allowlist **in policy code**     │                 │
      EXEMPTIONS: Platform edits policy YAML (no labels)   │                 │
                                                          PER-OBJECT exceptions:
                                                          app teams propose entries via PR

Design choices
	•	Default enforce: one binding per policy, cluster-wide.
	•	Namespace exemptions: in code inside each policy—Platform only.
	•	Per-object exceptions: provided at runtime from ExceptionRegistry.

⸻

6) Data Model

6.1 ExceptionRegistry (Cluster-scoped CRD)
	•	Purpose: carry only per-object exceptions. No namespace-wide switches here.
	•	Shape (informal):
	•	exceptions[]: list of entries:
	•	id: string (e.g., EX-TEAM-001)
	•	ticket: string (JIRA/Archer id, regex-validated in CI)
	•	ends: RFC3339 timestamp (CI ensures ≤ 60 days ahead)
	•	reason: short text
	•	rules[]: policyIds this entry applies to (noPrivileged, kafkaTopicNaming, …)
	•	namespace: target namespace
	•	selector: scope constraint
	•	labels (map, ANDed)
	•	names (list of exact resource names)
	•	kinds (optional list, e.g., ["Pod"])

Notes
	•	Requiring labels or names prevents blanket exemptions.
	•	rules[] lets one entry cover multiple related policies if needed.

⸻

7) Policy Authoring Standard

Each policy manifest is self-contained and follows the same structure.

Four fields you edit per policy
	1.	metadata.name (unique)
	2.	policyNsAllowlist (literal array, Platform-owned)
	3.	policyId (stable string keyed to business exceptions)
	4.	condition (CEL expression that evaluates true when the object is compliant)

Always include
	•	paramKind referencing ExceptionRegistry
	•	validations[0].expression = nsExempt || condition || excepted
	•	validationActions in Binding: start ["Warn","Audit"] then switch to ["Deny"]

Example excerpt (structure, not template):

variables:
- name: policyNsAllowlist
  expression: "['kube-system','legacy-migrate']"  # Platform edits here only
- name: nsExempt
  expression: "policyNsAllowlist.exists(ns, ns == request.namespace)"

- name: policyId
  expression: '"noPrivileged"'

- name: condition
  expression: |
    # CEL: TRUE means compliant
    !(exists(request.object.spec.containers, c,
        has(c.securityContext) && c.securityContext.privileged == true) ||
      (has(request.object.spec.initContainers) &&
       exists(request.object.spec.initContainers, c,
        has(c.securityContext) && c.securityContext.privileged == true)))

- name: excepted
  expression: |
    has(params.exceptions) &&
    exists(params.exceptions, e,
      e.namespace == request.namespace &&
      exists(e.rules, r, r == policyId) &&
      (!has(e.selector.kinds) || e.selector.kinds.exists(k, k == request.kind.kind)) &&
      (!has(e.selector.names) || (has(request.object.metadata.name) &&
                                  e.selector.names.exists(n, n == request.object.metadata.name))) &&
      (has(request.object.metadata.labels) &&
        (!has(e.selector.labels) ||
         e.selector.labels.all(k, v,
           has(request.object.metadata.labels[k]) &&
           request.object.metadata.labels[k] == v)))))
validations:
- expression: "nsExempt || condition || excepted"
  message: "Denied by policy:noPrivileged. Provide a scoped, time-boxed exception."


⸻

8) Governance & Security

8.1 RBAC
	•	Write to exceptionregistries.policyparams.yourorg.io: only GitOps SA (e.g., system:serviceaccount:flux-system:flux).
	•	Read can be granted cluster-wide for transparency.
	•	App teams have no write permissions to the CRD.

8.2 Separation of Duties
	•	Namespace exemptions are not in params; they’re embedded in policy code → only Platform can change them via PR to the policy file.
	•	Business exceptions are the only business-editable surface, validated by CI and review.

8.3 Audit
	•	PR history in both repos.
	•	Admission deny messages include policyId.
	•	API server audit logs can be forwarded to SIEM.

⸻

9) CI/CD

9.1 Business repo (vap-exceptions) CI
	•	Validate exceptions.yaml against JSON Schema.
	•	Enforce ends is valid RFC3339 and ≤ 60 days from now.
	•	Enforce selector contains labels or names.
	•	Optional: cap entries per namespace to avoid “pseudo-blanket” exemptions.

CI outcome: either pass → Platform CI can merge, or fail → feedback to team.

9.2 Platform repo (vap-constraints) CI
	•	Pull vap-exceptions at main.
	•	(Optionally) re-validate schema & expiry for defense-in-depth.
	•	Render/overwrite generated/exceptionregistry.combined.yaml with merged contents.
	•	GitOps applies the generated object.
	•	Policies and Bindings are applied from this repo.

Fail-closed behavior:
	•	parameterNotFoundAction: Deny in bindings ensures if the param is missing or malformed, we deny (safer default).

⸻

10) Operational Workflows

10.1 Adding a New Policy (Platform)
	1.	Copy an existing policy file.
	2.	Set:
	•	metadata.name
	•	policyNsAllowlist (start with system namespaces only; add case-by-case)
	•	policyId
	•	matchConstraints.resourceRules (correct GVR + operations)
	•	condition (CEL; true = compliant)
	3.	Create corresponding Binding:
	•	policyName set
	•	validationActions: ["Warn","Audit"] for warm-up
	•	paramRef.name: global-exception-registry
	•	parameterNotFoundAction: Deny
	4.	PR + review + merge → GitOps applies.
	5.	After observation period, flip to ["Deny"].

10.2 Business Exception Request (App Team)
	1.	Propose entry in vap-exceptions/registry-fragments/exceptions.yaml:
	•	ticket, reason, ends
	•	rules (policyIds)
	•	namespace
	•	selector.labels and/or selector.names
	2.	CI validates; Platform reviews; merge.
	3.	Platform CI regenerates exceptionregistry.combined.yaml; GitOps applies.
	4.	Exception is active until ends.

10.3 Removing/Extending Exceptions
	•	Remove: App team PR deletes the entry.
	•	Extend: New PR with new ends and justification/ticket.
	•	CI rejects excessive duration; Platform reviews.

10.4 Namespace-wide Exemption (Platform only)
	•	Edit the policy file; add namespace to policyNsAllowlist.
	•	PR + review + merge → GitOps applies.
	•	Revert later by removing from the list.

⸻

11) Runbooks

11.1 Troubleshooting “Why was my resource denied?”
	1.	Check the admission message → has policyId.
	2.	Locate policy YAML in vap-constraints/policies/.
	3.	Evaluate:
	•	Is namespace in policyNsAllowlist? (If so, it wouldn’t be denied.)
	•	Does the object satisfy condition? (Compare object fields to CEL condition.)
	•	Is there a matching exception in ExceptionRegistry?
	4.	If intended, create a scoped exception entry with ticket + expiry.

11.2 Debugging a Policy
	•	kubectl describe validatingadmissionpolicy <name> → conditions & CEL eval errors.
	•	kubectl apply --dry-run=server -f <object.yaml> → admission outcome without persisting.
	•	Verify the param exists: kubectl get exceptionregistry global-exception-registry -o yaml.

11.3 Param or Binding Missing
	•	Binding uses parameterNotFoundAction: Deny → all affected admissions will fail closed.
	•	Restore generated/exceptionregistry.combined.yaml via Platform CI; reconfirm GitOps health.

⸻

12) Observability & SLOs

Recommended
	•	Emit API server audit logs to log pipeline; tag events with policyId.
	•	Dashboards:
	•	Denies per policy over time
	•	Top namespaces / teams requesting exceptions
	•	Exception volume and average age; approaching expirations (N-day window)
	•	SLOs:
	•	Policy addition PR review time (Platform): target ≤ 2 business days.
	•	Exception PR turnaround (Platform): target ≤ 1 business day.
	•	Exception max duration: 30–60 days, org standard.

⸻

13) Risks & Mitigations

Risk	Impact	Mitigation
Over-reliance on exceptions	Policy dilution	CI caps duration; require ticket; review owner; dashboards & weekly review
Namespace exemptions drift	Hidden risk	Exemptions are in code (policies) with reviews; quarterly review of allowlists
Param unavailable	Admission fail-closed	Monitoring on GitOps health; quick CI rerun; small on-call playbook
CEL portability	Mis-eval across versions	Target 1.28+ semantics; keep expressions simple; test with dry-run
Large exception list	Performance/complexity	Keep entries narrow; prune expired; later consider byRule indexing if needed


⸻

14) Security & Compliance Notes
	•	Least privilege: only GitOps SA writes the param CR; app workloads cannot.
	•	No label/annotation switches: prevents stealth namespace bypasses.
	•	Auditability: all exemptions (ns-wide or object-level) leave a PR trail; exception entries include ticket and expiry.

⸻

15) Examples (Minimal, for reference)

ExceptionRegistry CRD (once, Platform):

apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: exceptionregistries.policyparams.yourorg.io
spec:
  group: policyparams.yourorg.io
  names:
    kind: ExceptionRegistry
    plural: exceptionregistries
  scope: Cluster
  versions:
  - name: v1alpha1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          exceptions:
            type: array
            items:
              type: object
              required: ["id","ticket","ends","rules","namespace","selector","reason"]
              properties:
                id: { type: string }
                ticket: { type: string }
                ends: { type: string }
                reason: { type: string, maxLength: 512 }
                rules: { type: array, items: { type: string } }
                namespace: { type: string }
                selector:
                  type: object
                  properties:
                    labels: { type: object, additionalProperties: { type: string } }
                    names: { type: array, items: { type: string } }
                    kinds: { type: array, items: { type: string } }

Policy (No Privileged Pods) (Platform):

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: pod-no-privileged
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["pods"]
  paramKind:
    apiVersion: policyparams.yourorg.io/v1alpha1
    kind: ExceptionRegistry
  variables:
  - name: policyNsAllowlist
    expression: "['kube-system','cattle-system','platform-sandbox','legacy-migrate']"
  - name: nsExempt
    expression: "policyNsAllowlist.exists(ns, ns == request.namespace)"
  - name: policyId
    expression: '"noPrivileged"'
  - name: condition
    expression: |
      !( exists(request.object.spec.containers, c,
            has(c.securityContext) && c.securityContext.privileged == true)
         ||
         (has(request.object.spec.initContainers) &&
          exists(request.object.spec.initContainers, c,
            has(c.securityContext) && c.securityContext.privileged == true)) )
  - name: excepted
    expression: |
      has(params.exceptions) &&
      exists(params.exceptions, e,
        e.namespace == request.namespace &&
        exists(e.rules, r, r == policyId) &&
        (!has(e.selector.kinds) || e.selector.kinds.exists(k, k == request.kind.kind)) &&
        (!has(e.selector.names) || (has(request.object.metadata.name) &&
                                    e.selector.names.exists(n, n == request.object.metadata.name))) &&
        (has(request.object.metadata.labels) &&
          (!has(e.selector.labels) ||
           e.selector.labels.all(k, v,
             has(request.object.metadata.labels[k]) &&
             request.object.metadata.labels[k] == v)))
      )
  validations:
  - expression: "nsExempt || condition || excepted"
    message: "Privileged containers are blocked (policy:noPrivileged). Use a scoped, time-boxed exception."

Binding (cluster-wide) (Platform):

apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: pod-no-privileged-binding
spec:
  policyName: pod-no-privileged
  validationActions: ["Deny"]   # start with ["Warn","Audit"] for dry run
  paramRef:
    name: global-exception-registry
    parameterNotFoundAction: Deny

Business exception entry (App team PR):

exceptions:
  - id: EX-CLR-001
    ticket: ARCHER-123456
    ends: 2025-11-30T23:59:59Z
    rules: ["noPrivileged"]
    namespace: dft-clearing-sys-01
    selector:
      kinds: ["Pod"]
      labels: { app: "legacy-ingestor" }
    reason: "Vendor tool requires CAP_SYS_ADMIN during migration"


⸻

16) Change Management & Communication
	•	Changelog in vap-constraints/README.md for policy updates and rationale.
	•	Monthly metrics review: exception count, average duration, top policies, namespaces.
	•	Quarterly policy review: trim namespace allowlists; retire exceptions that linger.

⸻

17) Future Enhancements (Roadmap)
	•	Optional byRule map in ExceptionRegistry for O(1) lookups at very large scale.
	•	Automated “expiry janitor” CronJob to flag/remove expired exceptions and notify owners.
	•	Policy library with re-usable CEL snippets (documentation only, still copy-paste per policy).
	•	Developer self-service portal to draft exception PRs with guardrails.

⸻

Final Notes
	•	This design intentionally favors clarity and governance over templating complexity.
	•	Duplication is acceptable: copy an existing policy and change the four fields.
	•	Namespace exemptions are only changeable by Platform through policy code—exactly matching your control requirements.
	•	The business experience is a single small YAML with ticket, reason, end date, and a narrow selector.

If you want, I can turn this into a polished PDF/Confluence-style document with diagrams and checklists your team can publish internally.
