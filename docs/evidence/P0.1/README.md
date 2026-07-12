# Evidence Manifest — P0.1

## CTX-P01-RV-001 — زمینه بازاعتبارسنجی مشترک

این Context تمام فیلدهای مشترک EV-P01-001 تا EV-P01-005 و EV-P01-007 را
تعریف می‌کند و هر رکورد با ارجاع صریح به آن، این مقادیر را به ارث می‌برد:

- Requirement: قرارداد Evidence و بازاعتبارسنجی Gate فاز P0.1؛
- Verification time UTC: <code>2026-07-12T21:16:56.5127772Z</code>؛
- Environment: Local repository documentation only؛ هیچ AD، IIS، SQL،
  Certificate، Integration یا Production لمس نشده است؛
- Working directory: <code>&lt;REPO_ROOT&gt;</code>؛
- Branch: <code>docs/p0-1-bootstrap</code>؛
- Verified predecessor commit:
  <code>0f5eaa81442cd602fb151bc38518295b76e87ef8</code>؛
- Verification target: working tree اصلاحات PR شماره ۱ که با پیام
  <code>docs: address PR review findings</code> commit خواهد شد؛ Hash نهایی
  از Git history حل می‌شود؛
- Tool versions:
  - <code>git version 2.55.0.windows.2</code>؛
  - <code>Windows PowerShell 5.1.26100.8737</code>؛
  - <code>ripgrep 15.1.0</code>؛
- Primary reviewer: Primary Codex agent؛
- Redaction: خروجی خام حساس ذخیره نشده و فقط خلاصه پاک‌سازی‌شده ثبت شده است.

## EV-P01-001 — وضعیت اولیه Repository

- Requirement/Work item: <code>P01-WI-01</code>؛ اتصال Repository، شاخه و
  وضعیت اولیه؛
- Context: <code>CTX-P01-RV-001</code>؛
- Original event date: 2026-07-12؛ زمان دقیق UTC رخداد اولیه ثبت نشده بود و
  در زمان UTC موجود در Context بازاعتبارسنجی شد؛
- Exact method:
  - <code>git remote -v</code>؛
  - <code>git status --short --branch</code>؛
  - <code>git log --oneline --decorate -5</code>؛
  - <code>git ls-remote --heads origin refs/heads/docs/p0-1-bootstrap</code>؛
- Exit code: <code>0</code>؛
- Relevant Git state:
  - Initial commit: <code>6b976b4</code>؛
  - Bootstrap commit: <code>0f5eaa8</code>؛
  - Branch: <code>docs/p0-1-bootstrap</code>؛
- Sanitized summary: remote برابر
  <code>https://github.com/milades/eportal.git</code> است و local/remote
  branch پیش از اصلاحات PR همگام و working tree تمیز بود؛
- Reviewer: Primary Codex agent؛
- Result: <code>PASS</code>.

## EV-P01-002 — یکسانی نقشه راه

- Requirement/Work item: <code>P01-WI-02</code> و <code>P01-AC-01</code>؛
- Context: <code>CTX-P01-RV-001</code>؛
- Exact method:
  - <code>Get-FileHash -Algorithm SHA256 docs\architecture\roadmap.md</code>؛
  - مقایسه دقیق با SHA-256 ثبت‌شده در
    <code>docs/architecture/baseline-metadata.md</code> و
    <code>APP-P0.1-001</code>؛
- Exit code: <code>0</code>؛
- Relevant Git state: فایل منجمد roadmap در commit
  <code>0f5eaa8</code> و working tree اصلاحات PR بدون تغییر محتوایی؛
- Sanitized summary: هر دو مقدار برابر
  <code>541968E579A8075845FCA161E8FED80A044A4EFE3182BF7B932C744673005EB6</code>
  هستند؛
- Reviewer: Primary Codex agent؛
- Result: <code>PASS</code>.

## EV-P01-003 — تأیید مالک

- Requirement/Work item: <code>P01-WI-04</code> تا <code>P01-WI-09</code> و
  <code>P01-AC-02</code> تا <code>P01-AC-07</code>؛
- Context: <code>CTX-P01-RV-001</code>؛
- Approval reference: <code>APP-P0.1-001</code>؛
- Decision date: 2026-07-12؛ زمان دقیق UTC پیام تصمیم در Artifact اولیه
  ذخیره نشده بود؛ اعتبار Artifact در زمان UTC Context بررسی شد؛
- Environment: Governance record in the Public repository؛
- Exact review method:
  - تطبیق SHA-256 سند با Approval؛
  - کنترل علامت‌خوردن هر هشت تصمیم؛
  - کنترل پذیرش Scope، HA، Backup و Monitoring؛
  - کنترل Ownerهای Security Review، Capacity Gate و UAT؛
- Exit code: <code>N/A</code> — تصمیم انسانی و بازبینی معنایی، نه فرمان shell؛
- Relevant Git state: Approval Artifact موجود در commit
  <code>0f5eaa8</code>؛
- Sanitized summary: Scope Baseline پذیرفته، Repository عمومی و سه ریسک
  خارج از Scope صریحاً پذیرفته شده‌اند؛ Project Owner فعلاً مالک سه Gate است؛
- Reviewer: Project Owner؛ بازاعتبارسنجی توسط Primary Codex agent؛
- Result: <code>PASS</code>.

## EV-P01-004 — اعتبارسنجی Bootstrap

- Requirement/Work item: <code>P01-WI-02</code>، <code>P01-WI-03</code> و
  کنترل کیفیت Bootstrap؛
- Context: <code>CTX-P01-RV-001</code>؛
- Exact method:
  - <code>git diff --check</code>؛
  - <code>rg -n '[ \t]+$' -g '*.md' -g '.gitignore' -g '.gitattributes' .</code>؛
  - اسکن <code>rg</code> برای Private Key header، GitHub token، PAT، access key،
    Password assignment، client_secret و refresh_token؛
  - <code>Get-FileHash -Algorithm SHA256 docs\architecture\roadmap.md</code>؛
  - شمارش شناسه‌های یکتای <code>P01-AC-01..07</code> و
    <code>P02-AC-01..24</code>؛
  - <code>rg --files -g '*.cs' -g '*.csproj' -g '*.sln' -g '*.slnx'</code>؛
- Exit code: <code>0</code>؛
- Relevant Git state: working tree اصلاحات PR روی predecessor
  <code>0f5eaa8</code>؛
- Sanitized summary: whitespace، Secret pattern، Hash، تعداد معیارها و نبود
  کد محصول بررسی شد؛ خروجی خام Secret scan ذخیره نشد؛
- Reviewer: Primary Codex agent؛
- Result: <code>PASS</code>.

## EV-P01-005 — بستن Gate و انتقال کنترل‌شده

- Requirement/Work item: تمام <code>P01-WI-01..09</code> و
  <code>P01-AC-01..07</code>؛
- Context: <code>CTX-P01-RV-001</code>؛
- Original verification time UTC:
  <code>2026-07-12T20:22:37.6939683Z</code>؛
- Exact method:
  - <code>git status --short --branch</code>؛
  - <code>git diff --check</code>؛
  - <code>Get-FileHash -Algorithm SHA256 docs\architecture\roadmap.md</code>؛
  - کنترل هشت تصمیم <code>APP-P0.1-001</code>؛
  - کنترل نبود Work Item یا Acceptance Criterion باز در P0.1؛
  - کنترل وضعیت‌های <code>Approved</code>، <code>Accepted</code> و
    <code>P0.2/Draft</code>؛
  - کنترل محدودیت دائمی Public repository و نبود کد محصول؛
- Exit code: <code>0</code>؛
- Relevant Git state: pre-bootstrap working tree که سپس در commit
  <code>0f5eaa8</code> ثبت شد و در Context فعلی بازاعتبارسنجی شده است؛
- Sanitized summary: P0.1 بسته و P0.2 فقط برای planning در وضعیت Draft فعال
  شد؛ Execution Approval همچنان NONE است؛
- Reviewer: Primary Codex agent و Independent governance agent؛
- Result: <code>PASS</code>.

## EV-P01-006 — بازبینی مستقل حاکمیتی

- Requirement/Work item: بازبینی مستقل lifecycle، Approval، Public repository
  و مجوزهای اثر خارجی برای P0.1؛
- Verification time UTC: <code>2026-07-12T20:25:42.9784247Z</code>؛
- Environment: Local repository documentation only؛
- Working directory: <code>&lt;REPO_ROOT&gt;</code>؛
- Tool/runtime: Independent Codex governance agent؛ نسخه runtime توسط App
  در خروجی بازبینی ارائه نشده است؛
- Relevant Git state: pre-bootstrap working tree مبتنی بر
  <code>6b976b4</code> که سپس بدون تغییر محتوایی در
  <code>0f5eaa8</code> ثبت شد؛
- Exact review method: بازبینی فقط‌خواندنی <code>AGENTS.md</code>،
  baseline metadata، Approval، status، بسته‌های P0.1/P0.2 و
  <code>git status</code> برای سازگاری transition و منع infra/commit/push؛
- Exit code: <code>N/A</code> — بازبینی معنایی عامل مستقل؛
- Sanitized summary: P0.1 Accepted، P0.2 Active/Draft، Execution Approval
  برابر NONE و محدودیت Public repository سازگار ارزیابی شدند؛
- Reviewer: Independent governance agent؛
- Result: <code>PASS</code>.

## EV-P01-007 — اصلاح یافته‌های بازبینی PR شماره ۱

- Requirement/Work item: رفع هشت finding بازبینی PR شماره ۱ بدون تغییر Scope،
  کد محصول یا زیرساخت؛
- Context: <code>CTX-P01-RV-001</code>؛
- Verification time UTC: <code>2026-07-12T21:16:56.5127772Z</code>؛
- Exact method:
  - کنترل SHA-256 roadmap؛
  - <code>git diff --check</code>؛
  - Secret و private-artifact scan؛
  - absolute user-path و email scan؛
  - relative Markdown link validation؛
  - شمارش ۲۴ معیار P0.2 و کنترل Gateهای Public Inventory، NTLM، IIS/gMSA،
    Integration isolation و SQL TLS؛
  - دو بازبینی مستقل governance و P0.2؛
  - بازبینی GitHub PR metadata و تطبیق local/remote SHA پس از Push؛
- Exit code: <code>0</code> برای مجموعه فرمان‌های verification؛
  <code>N/A</code> برای دو بازبینی معنایی مستقل؛
- Relevant Git state: working tree اصلاحات روی predecessor
  <code>0f5eaa8</code>؛ target همان commit حاوی این manifest است؛
- Sanitized summary: هر هشت finding رفع شد؛ roadmap SHA-256 ثابت، diff،
  Secret/private-artifact/path/email/link checks پاک، تعداد معیارهای P0.2
  برابر ۲۴ و تمام Gateهای Public Inventory، NTLM، IIS/gMSA، Integration
  isolation و SQL TLS حاضر هستند؛ هیچ کد محصول یا اثر زیرساختی ایجاد نشد؛
- Reviewer: Primary Codex agent، Independent governance agent و Independent
  P0.2/roadmap reviewer؛
- Result: <code>PASS</code>.
