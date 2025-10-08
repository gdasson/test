```apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: pod-no-privileged-binding
spec:
  policyName: pod-no-privileged
  validationActions: ["Deny"]   # start with ["Warn","Audit"] for dry run
  paramRef:
    name: global-exception-registry
    parameterNotFoundAction: Deny```
