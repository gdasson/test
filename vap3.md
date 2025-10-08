```variables:
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
  message: "Denied by policy:noPrivileged. Provide a scoped, time-boxed exception."```
