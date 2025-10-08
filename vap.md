Awesome — here’s a clean, copy-pasteable final design with no templating. Each policy is a self-contained YAML you can duplicate and tweak (policy ID, GVR, condition, namespace allowlist). Business users only touch the ExceptionRegistry file (per-object exemptions). Platform namespace exemptions are embedded in each policy.

⸻

VAP Exception Framework — Final Design (No Templating)

Audience: Platform/SRE & App Platform Engineers
Cluster: Kubernetes ≥ 1.26 (you’re on RKE2 1.31.x)
Principles:
	•	Enforce cluster-wide by default (one binding per policy, no labels/annotations).
	•	Namespace exceptions for Platform: embedded inside each policy (literal allowlist variable).
	•	Per-object exceptions for business teams: single cluster-scoped param (ExceptionRegistry) with time-boxed, narrowly scoped entries.
	•	Fail-closed if params missing.
	•	GitOps for everything (Flux/Argo), CI guardrails, and strict RBAC.

⸻

1) Repositories

1.1 vap-constraints (Platform-owned & applied)

vap-constraints/
  README.md
  crds/
    exceptionregistry.crd.yaml
  policies/                      # duplicate per policy, edit 4 fields
    pod-no-privileged.vap.yaml
    kafka-topic-standards.vap.yaml
    confluent-rolebinding.vap.yaml
    ... (add more policies here)
  bindings/
    pod-no-privileged.vapb.yaml
    kafka-topic-standards.vapb.yaml
    confluent-rolebinding.vapb.yaml
  generated/
    exceptionregistry.combined.yaml    # CI writes this from business repo input
  .github/workflows/
    merge-registry.yml
  CODEOWNERS

1.2 vap-exceptions (Business-owned; not applied directly)

vap-exceptions/
  registry-fragments/
    exceptions.yaml                # ONLY per-object exceptions
  schema/
    exceptions.schema.json         # JSON Schema for CI
  .github/workflows/
    validate.yml                   # schema + expiry checks
  README.md
  CODEOWNERS

Flux/Argo watches vap-constraints/ only. Business repo feeds the Platform CI, which renders generated/exceptionregistry.combined.yaml.

⸻

2) API: Cluster-scoped ExceptionRegistry CRD (param for business exceptions)

# vap-constraints/crds/exceptionregistry.crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: exceptionregistries.policyparams.yourorg.io
spec:
  group: policyparams.yourorg.io
  names:
    kind: ExceptionRegistry
    plural: exceptionregistries
    singular: exceptionregistry
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
            description: Per-object, time-boxed exceptions from business units.
            type: array
            items:
              type: object
              required: ["id","ticket","ends","rules","namespace","selector","reason"]
              properties:
                id:       { type: string }
                ticket:   { type: string }                    # ARCHER-1234 / JIRA-123
                ends:     { type: string }                    # RFC3339
                reason:   { type: string, maxLength: 512 }
                rules:    { type: array, items: { type: string } } # e.g., ["noPrivileged"]
                namespace:{ type: string }
                selector:
                  type: object
                  properties:
                    kinds:  { type: array, items: { type: string } }
                    names:  { type: array, items: { type: string } }
                    labels: { type: object, additionalProperties: { type: string } }

No namespace exemptions here. Those live inside each policy YAML.

⸻

3) Business input (simple & narrow)

# vap-exceptions/registry-fragments/exceptions.yaml
exceptions:
  - id: EX-CLR-001
    ticket: ARCHER-123456
    ends: 2025-11-30T23:59:59Z
    rules: ["noPrivileged"]
    namespace: dft-clearing-sys-01
    selector:
      kinds:  ["Pod"]
      labels: { app: "legacy-ingestor" }
    reason: "Vendor tool requires CAP_SYS_ADMIN during migration"

Rules:
	•	Must specify namespace.
	•	Must specify labels or names (can also include kinds).
	•	Must have ticket, reason, and end date (CI enforced).

⸻

4) Platform CI: merge to a single applied param

Produces the only param object the bindings will use:

# vap-constraints/generated/exceptionregistry.combined.yaml
apiVersion: policyparams.yourorg.io/v1alpha1
kind: ExceptionRegistry
metadata:
  name: global-exception-registry
exceptions:
  - id: EX-CLR-001
    ticket: ARCHER-123456
    ends: 2025-11-30T23:59:59Z
    rules: ["noPrivileged"]
    namespace: dft-clearing-sys-01
    selector:
      kinds:  ["Pod"]
      labels: { app: "legacy-ingestor" }
    reason: "Vendor tool requires CAP_SYS_ADMIN during migration"

Flux/Argo applies generated/exceptionregistry.combined.yaml.

⸻

5) Policies (duplicate & edit per policy)

Each policy file is self-contained. You’ll edit only four things:
	1.	metadata.name
	2.	policyNsAllowlist (namespaces Platform exempts for this policy)
	3.	policyId (short ID referenced by business exceptions)
	4.	condition (CEL: true means compliant)

5.1 Policy: No Privileged Pods

# vap-constraints/policies/pod-no-privileged.vap.yaml
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
  # (A) Platform namespace exemptions for THIS policy (literal list)
  - name: policyNsAllowlist
    expression: "['kube-system','cattle-system','platform-sandbox','legacy-migrate']"
  - name: nsExempt
    expression: "policyNsAllowlist.exists(ns, ns == request.namespace)"

  # (B) policy ID used by business exceptions
  - name: policyId
    expression: '"noPrivileged"'

  # (C) compliance condition (true = compliant)
  - name: condition
    expression: |
      !( exists(request.object.spec.containers, c,
            has(c.securityContext) && c.securityContext.privileged == true)
         ||
         (has(request.object.spec.initContainers) &&
          exists(request.object.spec.initContainers, c,
            has(c.securityContext) && c.securityContext.privileged == true)) )

  # (D) per-object exceptions from ExceptionRegistry
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

5.2 Policy: Kafka Topic Standards (Strimzi)

# vap-constraints/policies/kafka-topic-standards.vap.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: kafka-topic-standards
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["kafka.strimzi.io"]
      apiVersions: ["v1beta2"]
      operations: ["CREATE","UPDATE"]
      resources: ["kafkatopics"]

  paramKind:
    apiVersion: policyparams.yourorg.io/v1alpha1
    kind: ExceptionRegistry

  variables:
  - name: policyNsAllowlist
    expression: "['kafka-infra','platform-sandbox']"
  - name: nsExempt
    expression: "policyNsAllowlist.exists(ns, ns == request.namespace)"

  - name: policyId
    expression: '"kafkaTopicNaming"'

  - name: condition
    expression: |
      re("^[a-z0-9]+-(dev|qa|prod)-[a-z0-9-]{2,40}$").matches(object.metadata.name) &&
      has(object.metadata.labels) &&
      has(object.metadata.labels["owner"]) &&
      has(object.metadata.labels["team"])

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
    message: "KafkaTopic does not meet naming/label standards (policy:kafkaTopicNaming)."

5.3 Policy: ConfluentRoleBinding Hygiene (example)

# vap-constraints/policies/confluent-rolebinding.vap.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: confluent-rolebinding
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["platform.confluent.io"]
      apiVersions: ["v1beta1","v1"]
      operations: ["CREATE","UPDATE"]
      resources: ["confluentrolebindings"]

  paramKind:
    apiVersion: policyparams.yourorg.io/v1alpha1
    kind: ExceptionRegistry

  variables:
  - name: policyNsAllowlist
    expression: "['platform-sandbox']"
  - name: nsExempt
    expression: "policyNsAllowlist.exists(ns, ns == request.namespace)"

  - name: policyId
    expression: '"roleBindingHygiene"'

  - name: condition
    expression: |
      re("^(User:|ServiceAccount:)[A-Za-z0-9._:-]+$").matches(object.spec.principal) &&
      (["DeveloperRead","DeveloperWrite","ResourceOwner","SystemAdmin"]
        .exists(r, r == object.spec.role)) &&
      !(
        object.spec.resource.patternType == "LITERAL" &&
        object.spec.resource.name == "*" &&
        object.spec.role != "DeveloperRead"
      ) &&
      has(object.metadata.annotations) && has(object.metadata.annotations["change.ticket"])

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
    message: "ConfluentRoleBinding violates hygiene requirements (policy:roleBindingHygiene)."


⸻

6) Cluster-wide bindings (one per policy)

# vap-constraints/bindings/pod-no-privileged.vapb.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: pod-no-privileged-binding
spec:
  policyName: pod-no-privileged
  validationActions: ["Deny"]          # use ["Warn","Audit"] for dry-run
  paramRef:
    name: global-exception-registry
    parameterNotFoundAction: Deny
---
# vap-constraints/bindings/kafka-topic-standards.vapb.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: kafka-topic-standards-binding
spec:
  policyName: kafka-topic-standards
  validationActions: ["Deny"]
  paramRef:
    name: global-exception-registry
    parameterNotFoundAction: Deny
---
# vap-constraints/bindings/confluent-rolebinding.vapb.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: confluent-rolebinding-binding
spec:
  policyName: confluent-rolebinding
  validationActions: ["Deny"]
  paramRef:
    name: global-exception-registry
    parameterNotFoundAction: Deny


⸻

7) RBAC (recommended)
	•	App teams: no write permissions to exceptionregistries.policyparams.yourorg.io.
	•	Flux/Argo SA: write permissions for that CRD.

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: exceptionregistry-writer }
rules:
- apiGroups: ["policyparams.yourorg.io"]
  resources: ["exceptionregistries"]
  verbs: ["get","list","watch","create","update","patch","delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: exceptionregistry-writer-flux }
subjects:
- kind: ServiceAccount
  name: flux
  namespace: flux-system
roleRef:
  kind: ClusterRole
  name: exceptionregistry-writer
  apiGroup: rbac.authorization.k8s.io

(Optional) read-only role for visibility:

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata: { name: exceptionregistry-readonly }
rules:
- apiGroups: ["policyparams.yourorg.io"]
  resources: ["exceptionregistries"]
  verbs: ["get","list","watch"]


⸻

8) Business repo CI (vap-exceptions/.github/workflows/validate.yml)
	•	Validate against JSON Schema.
	•	Block ends beyond 60 days.
	•	Require selector labels or names.

Keep CI simple: yq + jsonschema + date checks. (Shell snippet omitted here for brevity; I can add it if you want it pre-baked.)

⸻

9) Platform repo CI (vap-constraints/.github/workflows/merge-registry.yml)
	•	Check out this repo + vap-exceptions at main.
	•	Optional: re-validate business file.
	•	Write generated/exceptionregistry.combined.yaml with merged exceptions.
	•	Flux/Argo picks it up.

⸻

10) Rollout plan
	1.	Verify VAP is present:

kubectl api-resources | grep -i ValidatingAdmissionPolicy

	2.	Apply CRD.
	3.	Apply policies with validationActions: ["Warn","Audit"] for 1–2 sprints.
	4.	Apply bindings (paramRef points to global-exception-registry).
	5.	Wire CI → ensure generated/exceptionregistry.combined.yaml exists.
	6.	Flip bindings to ["Deny"] per policy once false positives are cleared.
	7.	Onboard teams (README + example exception).

⸻

11) Operability (day-2)
	•	Smoke test (Pods):
	•	Create privileged Pod in non-exempt namespace → Denied with message.
	•	Add exception entry → admitted until ends.
	•	Create same in a policy-exempt namespace (legacy-migrate) → Admitted.
	•	Dry-run:

kubectl apply --dry-run=server -f object.yaml

	•	Debug CEL:

kubectl describe validatingadmissionpolicy <name>

	•	Guardrails: because namespace exemptions are only in the policy files, business teams cannot blanket-exempt via params.

⸻

12) Naming conventions & hygiene
	•	Policy name: (<domain>-)<rule> e.g., pod-no-privileged, kafka-topic-standards.
	•	policyId: stable, short ID used in exceptions.rules (e.g., noPrivileged, kafkaTopicNaming, roleBindingHygiene).
	•	Embed a comment header at the top of each policy with: owner, contact, and intended scope.
	•	Keep policyNsAllowlist minimal; review quarterly.
	•	Messages should mention policyId so users know which rule blocked them.

⸻

13) Adding a new policy (copy & edit checklist)
	1.	Copy any existing .vap.yaml into policies/<new-policy>.vap.yaml.
	2.	Edit:
	•	metadata.name
	•	policyNsAllowlist (literal list; start with only system namespaces if needed)
	•	policyId (string used by business entries)
	•	matchConstraints.resourceRules to the correct GVR + operations
	•	condition to your rule’s logic
	3.	Create matching binding (bindings/<new-policy>.vapb.yaml) pointing to global-exception-registry.
	4.	Commit to main → Flux/Argo applies.
	5.	Document policyId in the business README so teams know what to reference.

⸻

14) FAQ

Q: Can a business unit exempt a whole namespace?
A: No. Params do not contain namespace exemptions. Only Platform can exempt namespaces by editing the policy’s embedded list.

Q: What if we need a temporary full-namespace exemption?
A: Platform adds the namespace to policyNsAllowlist for that policy and later removes it. This is auditable via PR history.

Q: How do we avoid policy drift across hundreds of files?
A: Use a short internal checklist (section 13). Copy an existing policy and change only the four fields. Periodic code review can catch drift.

Q: Performance with many exceptions?
A: Keep exceptions narrow and short-lived. If volume grows large, consider sharding by rule (a byRule map) later; current flat list works well to start.

⸻

15) TL;DR
	•	Default enforce: cluster-wide binding per policy.
	•	Platform namespace exemptions: inside each policy YAML (literal allowlist).
	•	Business exceptions: per-object entries in a single ExceptionRegistry param (ticket, reason, end date, and selector required).
	•	No labels/annotations, no namespace off-switch in params.
	•	Duplicate policies are fine: edit policyId, GVR, condition, and embedded allowlist.

If you want, I can turn this into a zipped starter with the files above (ready to commit) and a brief onboarding README for app teams.
