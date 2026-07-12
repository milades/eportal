# راهنمای اجرای مرحله‌ای پروژه با عامل هوش مصنوعی

## 1. اصل راهبردی

سند نقشه راه نباید با یک دستور کلی مانند «همه پروژه را پیاده‌سازی کن» به مدل داده شود. اجرای قابل کنترل باید به شکل زیر باشد:

1. سند اصلی در مخزن پروژه به‌عنوان منبع حقیقت ثبت شود.
2. قواعد ثابت پروژه در فایل ریشه `AGENTS.md` قرار گیرد.
3. در هر مقطع فقط یک فاز فعال باشد.
4. برای فاز فعال، یک بسته اجرایی مستقل تهیه و تأیید شود.
5. عامل فقط همان فاز را پیاده‌سازی، آزمایش و مستندسازی کند.
6. پس از بازبینی و عبور از دروازه پذیرش، فاز بعدی فعال شود.

این روش باعث می‌شود محدوده، تصمیم‌های معماری، وضعیت اجرا و شواهد آزمون فقط به حافظه گفت‌وگو وابسته نباشند.

## 2. عامل مناسب

برای پیاده‌سازی واقعی، عامل هوش مصنوعی باید حداقل به این قابلیت‌ها دسترسی داشته باشد:

- خواندن و ویرایش فایل‌های مخزن؛
- اجرای PowerShell، `dotnet`، آزمون‌ها و ابزارهای Git؛
- مشاهده خروجی ساخت و آزمون؛
- نگهداری تغییرات در یک شاخه یا worktree جدا؛
- امکان توقف در دروازه‌های تأیید انسانی.

یک مدل گفت‌وگویی بدون دسترسی به مخزن و ترمینال فقط می‌تواند کد پیشنهادی تولید کند و برای اجرای قابل اعتماد صفر تا صد مناسب نیست.

## 3. ساختار پیشنهادی مخزن

    /
      AGENTS.md
      README.md
      docs/
        architecture/
          roadmap.md
        phases/
          P0.2-integration-preflight.md
          P1-solution-foundation.md
          P2-persistence-audit.md
          ...
        implementation/
          status.md
        adr/
        evidence/
          P0.2/
          P1/
          ...
      src/
      tests/

نسخه نهایی سند حاضر باید با نام `docs/architecture/roadmap.md` در همین مخزن کپی شود. فایل اصلی تغییرات معماری و محدوده را نگه می‌دارد؛ جزئیات اجرایی هر فاز در `docs/phases` قرار می‌گیرد.

## 4. الگوی فایل AGENTS.md

این فایل باید کوتاه، صریح و شامل قواعد پایدار باشد:

    # Project operating contract

    ## Sources of truth
    - Architecture and approved scope: docs/architecture/roadmap.md
    - Current execution state: docs/implementation/status.md
    - Active phase packet: path recorded in status.md

    ## Mandatory workflow
    - Before changing code, read all three sources of truth.
    - Implement only the active phase and its approved tasks.
    - Do not start the next phase without explicit user approval.
    - If code and documentation conflict, stop and report the conflict.
    - Record material architectural decisions as an ADR before implementation.
    - Never place real passwords, domain credentials, tokens, certificates, or
      production connection strings in source control, prompts, logs, or tests.
    - Do not change production AD, DNS, SPN, GPO, IIS, SQL Server, or certificate
      state without explicit approval for that exact operation.
    - Preserve unrelated user changes in the worktree.

    ## Quality gate
    - Build the affected solution.
    - Run the smallest relevant tests first, then the required phase suite.
    - Report every command and its result.
    - Update docs/implementation/status.md and phase evidence.
    - Stop at the phase exit gate and report PASS, FAIL, or BLOCKED.

    ## Technology invariants
    - .NET 10, C#, Blazor Web App with Static SSR authentication pages.
    - EF Core and SQL Server; hosting on Windows Server and IIS.
    - AD SSO is Kerberos-only; NTLM is not an accepted SSO success path.
    - Application authentication uses the approved JWT, refresh-token, and
      server-session design in the roadmap.
    - Phase-one exclusions remain excluded unless the roadmap is formally revised.

در صورت بزرگ شدن مخزن، می‌توان برای پوشه‌های حساس مانند `src/Security` یا `infra` فایل `AGENTS.md` محدودتر و نزدیک‌تر به همان کد اضافه کرد.

## 5. الگوی بسته اجرایی هر فاز

هر فایل فاز باید این بخش‌ها را داشته باشد:

    # Phase Pn — Name

    Status: Draft | Approved | In Progress | Review | Accepted | Blocked

    ## Objective
    یک نتیجه قابل اندازه‌گیری، نه صرفاً فهرست فعالیت‌ها.

    ## In scope
    موارد مجاز این فاز.

    ## Out of scope
    مواردی که عامل حق انجام آنها را ندارد.

    ## Preconditions
    پیش‌نیازها و شواهد لازم پیش از شروع.

    ## Approved decisions and inputs
    مقادیر واقعی یا فرض‌های تأییدشده.

    ## Work items
    - [ ] شناسه، شرح، خروجی و وابستگی هر کار

    ## Expected files/components
    فایل‌ها و اجزایی که احتمالاً ساخته یا تغییر داده می‌شوند.

    ## Tests and verification
    آزمون‌های واحد، یکپارچه، امنیتی و عملیاتی لازم.

    ## Evidence
    محل ثبت خروجی ساخت، آزمون، اسکرین‌شات یا گزارش تنظیمات.

    ## Rollback
    روش برگشت تغییرات همین فاز.

    ## Exit criteria
    معیارهای دودویی و قابل اثبات برای پذیرش فاز.

    ## Human approval gate
    چه کسی و بر اساس چه شواهدی فاز را می‌پذیرد.

## 6. الگوی فایل وضعیت اجرا

فایل `docs/implementation/status.md` باید در هر نوبت به‌روز شود:

    # Implementation status

    Active phase: P0.2
    Active packet: docs/phases/P0.2-integration-preflight.md
    Phase status: Draft
    Last verified commit: <hash or none>

    ## Completed
    - ...

    ## In progress
    - ...

    ## Blockers
    - ...

    ## Decisions made
    - ADR-...: ...

    ## Changed files
    - ...

    ## Verification evidence
    - Command: ...
      Result: PASS | FAIL
      Evidence: docs/evidence/...

    ## Next permitted action
    - ...

    ## Prohibited next action
    - Do not begin P1 until P0.2 is explicitly accepted.

این فایل برای ادامه کار در گفت‌وگوی جدید یا پس از قطع شدن نشست ضروری است.

## 7. پرامپت راه‌اندازی مخزن — بدون پیاده‌سازی

این دستور باید فقط یک بار و پس از قراردادن نقشه راه در مخزن اجرا شود:

    نقش تو عامل مهندسی ارشد این پروژه است. ابتدا این فایل‌ها را کامل بخوان:
    - AGENTS.md
    - docs/architecture/roadmap.md

    هدف این نوبت فقط آماده‌سازی کنترل‌شده پروژه برای اجرا است؛ هنوز کد محصول
    یا زیرساخت واقعی را پیاده‌سازی نکن.

    کارها:
    1. مخزن و وضعیت فعلی آن را بررسی کن.
    2. تناقض‌ها، ابهام‌های مسدودکننده و پیش‌نیازهای مفقود را گزارش کن.
    3. docs/implementation/status.md را ایجاد کن و P0.2 را فاز فعال قرار بده.
    4. از روی نقشه راه، بسته دقیق P0.2 را در docs/phases ایجاد کن.
    5. هر معیار پذیرش را دودویی و قابل اثبات بنویس.
    6. تغییرات بیرونی لازم در AD، DNS، SPN، GPO، IIS، SQL و گواهی را فقط به
       صورت چک‌لیست و فرمان پیشنهادی ثبت کن؛ هیچ‌کدام را اجرا نکن.

    خروجی نهایی باید شامل فایل‌های ایجادشده، ابهام‌ها، ریسک‌ها و پیشنهاد
    دقیق برای دروازه تأیید P0.2 باشد. پس از آن متوقف شو.

پس از بررسی خروجی، کاربر باید بسته P0.2 را صریحاً تأیید یا اصلاح کند.

## 8. پرامپت تحلیل و برنامه‌ریزی یک فاز — بدون تغییر کد

برای هر فاز ابتدا یک نوبت تحلیل اجرا شود:

    فایل‌های AGENTS.md، docs/architecture/roadmap.md،
    docs/implementation/status.md و بسته فاز فعال را کامل بخوان. سپس وضعیت
    Git و ساختار مخزن را بررسی کن.

    در این نوبت هیچ فایل محصول یا زیرساختی را تغییر نده. فقط:
    1. پوشش هر الزام فاز را به یک یا چند کار اجرایی نگاشت کن.
    2. ترتیب وابستگی‌ها و کوچک‌ترین واحدهای قابل آزمون را تعیین کن.
    3. فایل‌های مورد انتظار، آزمون‌ها، ریسک‌ها و rollback را مشخص کن.
    4. ابهام‌های واقعاً مسدودکننده را جدا کن؛ برای امور غیرمسدودکننده فرض
       محافظه‌کارانه پیشنهاد بده.
    5. یک جدول PASS/FAIL برای معیار خروج فاز ارائه کن.

    خارج از فاز فعال برنامه‌ریزی یا پیاده‌سازی نکن. در پایان منتظر تأیید بمان.

## 9. پرامپت اجرای فاز تأییدشده

پس از تأیید برنامه همان فاز:

    برنامه فاز فعال تأیید شد. مطابق AGENTS.md و فقط در محدوده بسته فاز فعال
    کار کن.

    الزامات اجرا:
    - پیش از ویرایش، وضعیت Git و تغییرات موجود را بررسی و حفظ کن.
    - کارها را به ترتیب وابستگی و در checkpointهای کوچک اجرا کن.
    - پس از هر واحد معنادار، ساخت یا کوچک‌ترین آزمون مرتبط را اجرا کن.
    - در صورت نیاز به تغییر معماری، محدوده، امنیت یا قرارداد عمومی، قبل از
      پیاده‌سازی متوقف شو و ADR پیشنهادی ارائه کن.
    - هیچ راز واقعی تولید یا ثبت نکن.
    - هیچ تغییری در محیط واقعی AD/DNS/SPN/GPO/IIS/SQL/Certificate بدون
      مجوز صریح همان عملیات انجام نده.
    - تمام فرمان‌ها و نتیجه آنها را در شواهد فاز ثبت کن.
    - docs/implementation/status.md را به‌روز نگه دار.

    در پایان:
    1. تغییرات و فایل‌ها را خلاصه کن.
    2. معیارهای خروج را با PASS/FAIL/BLOCKED و شاهد گزارش کن.
    3. آزمون‌های اجراشده و نتیجه دقیق آنها را گزارش کن.
    4. بدهی‌ها، ریسک‌های باقیمانده و rollback را بیان کن.
    5. در دروازه پذیرش متوقف شو و فاز بعدی را شروع نکن.

## 10. پرامپت بازبینی مستقل فاز

بهتر است بازبینی در یک نشست یا عامل جداگانه انجام شود:

    تغییرات فاز فعال را مستقل از عامل پیاده‌ساز بازبینی کن. منابع الزام:
    AGENTS.md، roadmap، status و بسته فاز فعال هستند.

    diff و کد کامل مرتبط را بررسی و آزمون‌های لازم را اجرا کن. تمرکز ویژه:
    - نقض محدوده یا معیار پذیرش؛
    - آسیب‌پذیری‌های احراز هویت، JWT، cookie، session، CSRF و logging؛
    - نشت credential، token یا داده حساس؛
    - خطاهای همزمانی، انقضا، revoke، account switching و failure mode؛
    - migration و سازگاری SQL؛
    - مسیر ناخواسته NTLM؛
    - نبود آزمون مؤثر یا آزمون ظاهراً سبز؛
    - رفتار fail-open در کنترل‌های امنیتی.

    ابتدا یافته‌ها را بر اساس شدت و با محل دقیق فایل گزارش کن. سپس وضعیت هر
    معیار خروج را PASS/FAIL/BLOCKED اعلام کن. قابلیت جدید اضافه نکن.

پس از رفع یافته‌ها، همین بازبینی و آزمون‌ها دوباره اجرا شوند.

## 11. پرامپت ادامه کار پس از قطع نشست

    به حافظه گفت‌وگوی قبلی تکیه نکن. ابتدا AGENTS.md، roadmap، status، بسته
    فاز فعال و git diff/status را بخوان. سپس:
    1. آخرین وضعیت اثبات‌شده را خلاصه کن.
    2. تغییرات ناقص یا بدون آزمون را مشخص کن.
    3. فقط از Next permitted action در status ادامه بده.
    4. پیش از هر تغییر جدید، کوچک‌ترین بررسی لازم برای صحت وضعیت را اجرا کن.
    5. در همان دروازه فاز متوقف شو؛ فاز بعدی را شروع نکن.

## 12. ترتیب اجرایی همین پروژه

ترتیب کلی باید از نقشه راه اصلی گرفته شود و به‌صورت یک فاز فعال در هر زمان اجرا گردد:

1. `P0.1` — تصویب رسمی سند و محدوده؛
2. `P0.2` — تعیین مقادیر واقعی، ایجاد Integration و preflight زیرساخت؛
3. `P1` — پایه Solution، معماری و تنظیمات؛
4. `P2` — persistence، migration و audit؛
5. `P3` — ورود دستی و اعتبارسنجی AD؛
6. `P4` — JWT، refresh token و server session؛
7. `P5` — SSO مبتنی بر Kerberos، logout و account switching؛
8. `P6` — رابط کاربری و جریان‌های کامل؛
9. `P7` — hardening و آزمون‌های امنیتی؛
10. `P8` — ظرفیت، بار و پایداری؛
11. `P9` — استقرار، rollback و تحویل نهایی.

اولین اقدام صحیح در وضعیت فعلی، تصویب `P0.1` و سپس تولید و تأیید بسته اجرایی `P0.2` است؛ شروع مستقیم کدنویسی پیش از P0.2 توصیه نمی‌شود.

## 13. قواعد کنترل کیفیت و تغییر

- هر الزام باید حداقل یک پیاده‌سازی و یک روش اثبات داشته باشد.
- «ساخته شد» با «پذیرفته شد» یکسان نیست؛ پذیرش فقط با شاهد و معیار خروج انجام می‌شود.
- تغییرات خارج از فاز باید به backlog بروند، نه اینکه پنهانی وارد کد شوند.
- هر تغییر معماری مهم ابتدا ADR و تأیید می‌خواهد.
- checkpoint یا commit باید کوچک و قابل برگشت باشد؛ commit تنها با مجوز کاربر انجام شود.
- شکست آزمون نباید حذف، skip یا با کاهش معیارها پنهان شود.
- اگر نیازمند دسترسی مدیریتی یا تغییر محیط واقعی است، عامل باید فرمان و اثر آن را توضیح دهد و برای همان عملیات مجوز بگیرد.
- اسناد status و evidence بخشی از خروجی فاز هستند، نه کار اختیاری.

## 14. تعریف خروجی قابل اعتماد هر فاز

یک فاز تنها زمانی آماده پذیرش است که هم‌زمان این موارد وجود داشته باشد:

- کد یا پیکربندی در محدوده مصوب؛
- ساخت موفق؛
- آزمون‌های لازم و نتیجه ثبت‌شده؛
- نگاشت الزام به شاهد؛
- نبود یافته بحرانی یا مهم باز؛
- rollback قابل اجرا؛
- status به‌روز؛
- بازبینی انسانی یا مستقل؛
- تأیید صریح دروازه فاز.
