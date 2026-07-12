# Evidence Manifest — P0.1

## EV-P01-001 — وضعیت اولیه Repository

- Date: 2026-07-12
- Environment: Local repository
- Working directory: <code>C:\Users\MILAD ESMAEILI\Documents\Codex\eportal</code>
- Method: <code>git remote -v</code>، <code>git status --short --branch</code> و <code>git log --oneline --decorate -5</code>
- Exit code: 0
- Remote: <code>origin https://github.com/milades/eportal.git</code>
- Initial branch: <code>main</code>
- Initial commit: <code>6b976b4</code>
- Initial status: clean and aligned with <code>origin/main</code>
- Working branch created: <code>docs/p0-1-bootstrap</code>
- Result: PASS

## EV-P01-002 — یکسانی نقشه راه

- Method: <code>Get-FileHash -Algorithm SHA256</code> روی فایل مبدأ و نسخه Repository
- Exit code: 0
- Source SHA-256: <code>541968E579A8075845FCA161E8FED80A044A4EFE3182BF7B932C744673005EB6</code>
- Repository SHA-256: <code>541968E579A8075845FCA161E8FED80A044A4EFE3182BF7B932C744673005EB6</code>
- Result: PASS

## EV-P01-003 — تأیید مالک

- Approval reference: <code>APP-P0.1-001</code>
- Date: 2026-07-12
- Actor: Project Owner
- Method: Explicit approval in the current Codex task
- Approved roadmap SHA-256: <code>541968E579A8075845FCA161E8FED80A044A4EFE3182BF7B932C744673005EB6</code>
- Repository visibility: Public
- Scope inside/outside phase one: Accepted
- No-HA risk: Accepted
- No-Backup risk: Accepted
- No-Monitoring risk: Accepted
- Security Review owner: Project Owner
- Capacity Gate owner: Project Owner
- UAT owner: Project Owner
- Result: PASS

## EV-P01-004 — اعتبارسنجی Bootstrap

- Date: 2026-07-12
- Environment: Local repository
- Methods:
  - <code>git diff --check</code>
  - اسکن trailing whitespace
  - اسکن الگوهای Private Key، GitHub token و Secret
  - کنترل SHA-256 نقشه راه
  - کنترل وجود ۷ معیار P0.1 و ۲۴ معیار P0.2
  - کنترل نبود فایل‌های cs، csproj، sln و slnx
- Exit code: 0
- Result:
  - Diff check: PASS
  - Trailing whitespace: PASS
  - Secret pattern scan: PASS
  - Roadmap integrity: PASS
  - Acceptance criteria count: PASS
  - No product code: PASS
- Overall result: PASS

## EV-P01-005 — بستن Gate و انتقال کنترل‌شده

- Date: 2026-07-12
- UTC verification time: <code>2026-07-12T20:22:37.6939683Z</code>
- Environment: Local repository documentation
- Working directory: <code>C:\Users\MILAD ESMAEILI\Documents\Codex\eportal</code>
- Tool versions:
  - <code>git version 2.55.0.windows.2</code>
  - <code>Windows PowerShell 5.1.26100.8737</code>
- Reviewer: Primary agent; followed by independent governance agent review
- Methods:
  - <code>git status --short --branch</code>
  - <code>git diff --check</code>
  - <code>Get-FileHash -Algorithm SHA256 docs\architecture\roadmap.md</code>
  - <code>rg</code> checks for approval boxes، open P0.1 criteria، lifecycle
    states، public-repository constraints، secret patterns and product-code files
- Exit code: 0
- Result:
  - Roadmap hash: PASS
  - P0.1 approval decisions: PASS
  - P0.1 exit criteria: PASS
  - P0.2 activation as Draft only: PASS
  - Public repository constraints: PASS
  - Secret pattern scan: PASS
  - No product code: PASS
- Overall result: PASS

## EV-P01-006 — بازبینی مستقل حاکمیتی

- UTC verification time: <code>2026-07-12T20:25:42.9784247Z</code>
- Reviewer: Independent governance agent
- Scope:
  - P0.1 Accepted و P0.2 Active/Draft؛
  - جداسازی Activation approval از Execution approval؛
  - منع دائمی topology واقعی در Repository عمومی؛
  - supersession برچسب‌های lifecycle سند منجمد؛
  - منع تغییر زیرساخت، commit و push بدون مجوز.
- Finding: No blocker remains
- Result: PASS
