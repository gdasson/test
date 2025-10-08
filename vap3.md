```exceptions:
  - id: EX-CLR-001
    ticket: ARCHER-123456
    ends: 2025-11-30T23:59:59Z
    rules: ["noPrivileged"]
    namespace: dft-clearing-sys-01
    selector:
      kinds: ["Pod"]
      labels: { app: "legacy-ingestor" }
    reason: "Vendor tool requires CAP_SYS_ADMIN during migration"```
