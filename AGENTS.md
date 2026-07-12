# EPortal Project Operating Contract

## Authoritative sources

1. Approved normative scope and architecture: <code>docs/architecture/roadmap.md</code>
2. Baseline approval metadata: <code>docs/architecture/baseline-metadata.md</code>
3. Human approvals: <code>docs/approvals/</code>
4. Current execution state: <code>docs/implementation/status.md</code>
5. Active phase packet: the exact path recorded in the status file

Read all applicable authoritative sources before planning or changing files.
A phase packet may refine execution but may not expand or contradict the roadmap.
On conflict, stop, record the conflict, and report <code>BLOCKED</code>.

The approved roadmap file is immutable at its recorded SHA-256. Its embedded
pre-approval status labels are historical and are superseded only for lifecycle
state by <code>APP-P0.1-001</code> and the baseline metadata file. This
supersession does not alter or override any normative scope, architecture,
security, risk, acceptance, or delivery requirement in the roadmap.

## Phase containment and authority

- Work only on the active phase and its recorded next permitted action.
- Do not start, scaffold, or partially implement a later phase.
- Phase states are Draft, Awaiting Owner Approval, Approved, In Progress,
  Review, Accepted, or Blocked.
- Only the project owner may set Approved or Accepted, change the active phase,
  or approve a scope/architecture change.
- An AI agent must never infer approval from silence or approve its own work.
- Record every approved transition with an approval reference.
- Put out-of-phase ideas in a proposed change or backlog; do not implement them.

## Repository safety

- Inspect Git status and diff before edits; preserve unrelated user changes.
- Work on a dedicated branch. Do not change <code>main</code> directly.
- Do not commit, push, tag, release, create a pull request, merge, or deploy
  unless the user explicitly requests that exact action.
- Do not rewrite Git history or delete user work.
- Keep text files UTF-8 and use deterministic, reviewable changes.

## External effects are default-deny

- Repository work does not authorize access to or mutation of AD, DNS, SPN,
  gMSA, GPO, IIS, SQL Server, certificates, Registry, Windows policy, or
  Production/Integration hosts.
- Before any real infrastructure operation, state the target, environment,
  exact command/action, expected effect, required privilege, evidence, and
  rollback. Execute it only after explicit approval for that operation.
- A general instruction such as “continue” is not infrastructure approval.
- Read-only probes also require an identified target and approval when their
  output can expose topology, account data, policy, or security events.
- Infrastructure scripts must separate Test from Set, be idempotent where
  possible, fail closed, guard the environment, and support WhatIf/confirmation.
- Creating a script never grants permission to run it.

## Secrets, credentials and privacy

- Never store or expose passwords, domain credentials, PATs, private keys/PFX,
  HMAC or Data Protection keys, JWTs, refresh tokens, cookies, credential-bearing
  connection strings, raw user data, or memory/database dumps.
- Use placeholders or secret references. Keep real secret values outside Git,
  prompts, source, tests, logs, artifacts, and evidence.
- Sanitize command output before recording it.
- Do not commit raw AD/Windows security logs or packet captures. Store only a
  redacted summary and, when needed, a hash and controlled external location.
- While repository visibility is Public, never commit real internal FQDNs,
  SPNs, gMSA names, database names, certificate thumbprints, IPs, or restricted
  topology. Use placeholders or non-sensitive references even after visibility
  has been confirmed.

## Technology and security invariants

- .NET 10, C#, Blazor Web App and Static SSR authentication pages.
- EF Core with SQL Server; Windows Server and IIS hosting.
- Chrome on managed, domain-joined Windows clients is the phase-one browser.
- SSO is Kerberos-only. NTLM is never an accepted SSO success path.
- Application authentication follows the approved JWT, refresh-token, and
  server-side session design in the roadmap.
- Authentication and infrastructure failures fail closed.
- HA, Backup, Monitoring, business authorization, MFA/federation, public API,
  internet access, and non-Chrome support remain outside phase-one scope.
- Changing these invariants requires a Change Request, impact analysis,
  an ADR where applicable, and explicit owner approval.

## Verification and evidence

- Every PASS must identify the requirement, exact command or review method,
  exit code, environment, relevant commit/state, and sanitized evidence.
- Use PASS, FAIL, BLOCKED, and NOT RUN distinctly.
- A skipped, deleted, or weakened test cannot be used to obtain PASS.
- Local or Fake tests cannot mark AD, Kerberos, IIS, SQL, Chrome GPO, or E2E
  Integration criteria as PASS.
- Run the smallest relevant verification first, then the required phase suite.
- Update the status file and evidence manifest before stopping.
- Stop at the phase gate and do not begin the next phase.
