```apiVersion: admissionregistration.k8s.io/v1
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
    message: "Privileged containers are blocked (policy:noPrivileged). Use a scoped, time-boxed exception."```
