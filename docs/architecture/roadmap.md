# سند محدوده، معماری و نقشه راه اجرایی

## پرتال داخلی سازمان با SSO اکتیو دایرکتوری و ورود دستی

| مشخصه | مقدار |
|---|---|
| نسخه سند | 1.0 |
| تاریخ مبنا | ۲۱ تیر ۱۴۰۵ / 12 July 2026 |
| وضعیت | خط مبنای پیشنهادی برای تأیید پیش از پیاده‌سازی |
| فاز مورد پوشش | فاز اول: احراز هویت، نشست و زیرساخت پایه پرتال |
| فناوری اصلی | .NET 10 LTS، C#، Blazor Web App با Static SSR، EF Core، SQL Server، IIS |
| روش استفاده از سند | منبع اجرای کار، کنترل تغییر محدوده و معیار پذیرش |

---

## 1. نتیجه اجرایی

خروجی فاز اول یک پرتال داخلی تحت وب است که کاربر سازمانی را از دو مسیر مستقل اما همگرا احراز هویت می‌کند:

1. ورود خودکار با حساب جاری Windows از طریق Kerberos و Windows Authentication؛
2. ورود دستی با نام کاربری ساده و رمز عبور دامین.

Windows Authentication فقط روی درگاه مخصوص SSO استفاده می‌شود. پس از اثبات هویت در هر دو مسیر، سامانه یک هویت محلی، نشست قابل ابطال در SQL Server، JWT کوتاه‌عمر و Refresh Token چرخشی ایجاد می‌کند. تمام صفحات محافظت‌شده بر اساس JWT و وضعیت نشست سامانه مجاز می‌شوند، نه بر اساس Windows Principal.

کاربر می‌تواند از سامانه خارج شود، بدون اینکه از Windows خارج شود. پس از Logout، SSO خودکار برای همان نشست Chrome متوقف می‌شود و فرم ورود دستی ظاهر می‌گردد. کاربر می‌تواند با حساب دامین دیگری وارد شود یا با انتخاب دکمه «ورود با حساب ویندوز» دوباره SSO را فعال کند.

این سند معماری، محدوده، مدل داده، جریان‌ها، پیش‌نیازهای زیرساخت، برنامه تست، استقرار، Rollback، برآورد و ریسک‌ها را به‌اندازه لازم برای شروع اجرای کنترل‌شده مشخص می‌کند.

.NET 10 یک نسخه LTS است و مطابق سیاست فعلی Microsoft تا 14 November 2028 پشتیبانی می‌شود؛ Production باید روی آخرین Patch پشتیبانی‌شده شاخه 10.0.x باقی بماند.

---

## 2. خط مبنا و فرض‌های مصوب

### 2.1 فرض‌های کسب‌وکاری و محیطی

| موضوع | خط مبنا |
|---|---|
| مخاطب | کاربران داخلی سازمان |
| شبکه | کاملاً داخلی و محلی |
| دامین | یک Microsoft Active Directory Domain |
| کلاینت رسمی | Windows عضو دامین با Google Chrome مدیریت‌شده |
| مرورگرهای دیگر | پشتیبانی و گواهی نمی‌شوند |
| سرور | یک Windows Server عضو همان دامین |
| میزبانی | IIS، مدل In-Process، یک Application Pool اختصاصی |
| پایگاه داده | SQL Server روی همان سرور IIS |
| ظرفیت اسمی | ۲٬۰۰۰ کاربر |
| اوج هم‌زمانی | ۱٬۰۰۰ کاربر |
| هدف Stress | ۱٬۵۰۰ نشست هم‌زمان |
| HA | خارج از محدوده |
| رابط | فارسی، راست‌چین، Chrome دسکتاپ |
| مدیر تنظیمات | فایل مرجع محیط به‌همراه اسکریپت‌های استقرار |

### 2.2 مقادیر فرضی قابل جایگزینی

این نام‌ها واقعی نیستند و پیش از شروع فاز زیرساخت باید جایگزین شوند:

| پارامتر | مقدار فرضی |
|---|---|
| AD DNS Domain | <code>corp.mydomain.ir</code> |
| AD NetBIOS | <code>MYDOMAIN</code> |
| Production FQDN | <code>portal.mydomain.ir</code> |
| Integration FQDN | <code>portal-int.mydomain.ir</code> |
| Production gMSA | <code>MYDOMAIN\gmsaPrtlProd$</code> |
| Integration gMSA | <code>MYDOMAIN\gmsaPrtlInt$</code> |
| Production Database | <code>PortalAuth</code> |
| Integration Database | <code>PortalAuth_Integration</code> |
| JWT Issuer | <code>https://portal.mydomain.ir</code> |
| JWT Audience | <code>portal-web</code> |
| منطقه زمانی نمایش | <code>Asia/Tehran</code> |
| زمان ذخیره‌سازی | UTC |

### 2.3 سیاست نشست مصوب

| سیاست | مقدار اولیه |
|---|---:|
| عمر Access JWT | ۹ دقیقه و ۳۰ ثانیه |
| حداکثر بی‌کاری نشست | ۳۰ دقیقه |
| حداکثر عمر مطلق نشست | ۸ ساعت |
| Clock Skew | ۳۰ ثانیه |
| بازاعتبارسنجی وضعیت AD | هنگام هر Refresh؛ سقف فنی ۱۰ دقیقه با احتساب Clock Skew |
| Grace هم‌زمانی Refresh | ۵ ثانیه |
| Logout | ابطال فوری نشست جاری |
| نشست‌های هم‌زمان یک کاربر | مجاز |
| Logout همه دستگاه‌ها | خارج از محدوده |

تمام مقادیر زمانی قابل تنظیم‌اند و با تغییر فایل محیط و Recycle کنترل‌شده IIS اعمال می‌شوند.

---

## 3. محدوده

### 3.1 داخل محدوده فاز اول

- ایجاد Solution و معماری Modular Monolith برای توسعه آینده پرتال؛
- Blazor Web App با صفحات احراز هویت Static SSR؛
- ورود خودکار Kerberos از Chrome مدیریت‌شده؛
- ورود دستی با <code>sAMAccountName</code> و رمز دامین؛
- بررسی وجود و نوع User، فعال‌بودن، قفل‌نبودن، منقضی‌نبودن حساب و رمز؛
- ساخت و همگام‌سازی پروفایل محلی حداقلی بدون ذخیره Credential؛
- JWT امضاشده داخل Cookie امن؛
- Refresh Token opaque، چرخشی و قابل تشخیص از نظر reuse؛
- نشست قابل ابطال و بررسی <code>sid</code> در SQL در هر درخواست محافظت‌شده؛
- Logout، توقف SSO خودکار و ورود دستی با حساب دیگر؛
- Audit امنیتی با نگهداری پیش‌فرض ۱۸۰ روز؛
- Rate Limit و محافظت در برابر Brute Force و Password Spraying؛
- تنظیمات Strongly Typed، Validate-on-start و فایل پارامتر مرجع؛
- اسکریپت‌های Preflight، نصب، پیکربندی، استقرار و Rollback؛
- محیط Integration حداقلی روی همان سرور با Site، App Pool، DB، FQDN، SPN و حساب‌های تست جدا؛
- تست‌های Unit، Integration، E2E، امنیت، خرابی و ظرفیت؛
- مستندات راه‌اندازی، عملیات تغییر نسخه و شواهد پذیرش.

### 3.2 خارج از محدوده فاز اول

- ماژول‌های تجاری پرتال؛
- Role، Group، OU filtering و Authorization تجاری؛
- ثبت‌نام، تغییر رمز، بازیابی رمز یا Unlock حساب؛
- MFA، ADFS، Entra ID، OIDC، SAML یا Federation؛
- دسترسی اینترنتی، موبایل، کلاینت خارج از دامین یا مرورگر غیر از Chrome؛
- مدیریت کاربران یا تنظیمات از طریق پنل مدیریتی؛
- Hot Reload تنظیمات؛
- API عمومی یا CORS؛
- NTLM به‌عنوان روش پذیرفته‌شده SSO؛
- Load Balancer، سرور دوم، Failover و High Availability؛
- Backup، Restore، RPO و RTO؛
- Health Check، داشبورد Monitoring، SIEM، Alerting و پایش انقضای گواهی؛
- اتصال به یک پلتفرم CI/CD خاص؛ اسکریپت‌های Build و Deploy مستقل تحویل می‌شوند؛
- Logout از Windows یا حذف Ticketهای Kerberos؛
- Logout همه نشست‌های یک کاربر؛
- پشتیبانی رسمی از دستگاه‌های غیرمدیریت‌شده.

### 3.3 ریسک‌های پذیرفته‌شده ناشی از خارج‌بودن موارد

- خرابی سرور واحد موجب توقف کامل IIS و SQL می‌شود.
- نبود Backup می‌تواند موجب ازبین‌رفتن پروفایل، Audit و نشست‌ها شود.
- نبود Monitoring ممکن است کشف خطا، پرشدن منابع یا انقضای گواهی را تا گزارش کاربر به تأخیر بیندازد.
- محیط Integration روی همان سرور، استقلال فیزیکی و آزمون Disaster Recovery فراهم نمی‌کند.
- پس از عملیاتی‌شدن Production، تست بار روی همان سرور مجاز نیست؛ تست ظرفیت باید پیش از Go-Live انجام شود.

---

## 4. معیارهای موفقیت فاز اول

فاز اول فقط زمانی کامل محسوب می‌شود که همه موارد زیر با شواهد قابل تکرار تأیید شوند:

1. کاربر فعال دامین در Chrome بدون ورود مجدد رمز و با Kerberos وارد شود.
2. پروتکل واقعی Kerberos باشد و NTLM پنهان به‌عنوان موفقیت پذیرفته نشود.
3. مسیرهای عادی وب هیچ Windows Challenge ایجاد نکنند؛ فقط endpoint مخصوص SSO مجاز باشد.
4. کاربر بتواند Logout کند و در همان Windows Session با حساب دامین دیگری دستی وارد شود.
5. دکمه «ورود با حساب ویندوز» SSO را دوباره فعال کند.
6. حساب ناموجود، Disabled، Locked، Account Expired، Password Expired و Must Change Password هیچ JWT دریافت نکند.
7. رمز عبور، JWT، Refresh Token و Cookie در SQL، Log، Audit، Claims، Trace یا مرورگر قابل مشاهده برای JavaScript نباشد.
8. Logout کپی قبلی JWT را به‌سبب ابطال <code>sid</code> فوراً نامعتبر کند.
9. قطع AD مانع Login و Refresh شود؛ قطع SQL تمام دسترسی محافظت‌شده را Fail Closed کند.
10. تغییر وضعیت حساب AD حداکثر در Refresh بعدی و در سقف AccessLifetime + ClockSkew، برابر ۱۰ دقیقه، اعمال شود.
11. سامانه ۱٬۰۰۰ Virtual User هم‌زمان و تست Stress برابر ۱٬۵۰۰ Virtual User را با معیارهای کارایی بخش 14 پاس کند.
12. Recycle IIS تنظیمات، Data Protection و امکان Login را به‌شکل غیرمنتظره خراب نکند.
13. Migration، استقرار و Rollback آزمایشی طبق Runbook انجام شوند.
14. تمام مقادیر فرضی پیش از Production با مقادیر واقعی جایگزین و Preflight موفق باشد.

---

## 5. معماری منطقی

### 5.1 نمای اجزا

    Domain-joined Windows + Chrome
                 |
                 | HTTPS / FQDN / Chrome GPO
                 v
       IIS + ASP.NET Core Module
          |                 |
          | JWT default     | Windows scheme only on SSO endpoint
          v                 v
        Blazor SSR      Kerberos / AD DS
          |
          +---- Application Services
          |       |-- Manual Authentication
          |       |-- Account Eligibility
          |       |-- Session / JWT / Refresh
          |       |-- Audit
          |
          +---- EF Core ---- SQL Server

### 5.2 مسئولیت اجزا

| جزء | مسئولیت |
|---|---|
| Chrome | ارسال خودکار Credential ویندوز فقط برای FQDN مجاز در GPO |
| IIS | TLS، میزبانی، Windows Authentication و انتقال Windows Identity فقط به مسیر SSO |
| Blazor Web | Static SSR، فرم‌ها، Redirect، Cookie، Anti-forgery و صفحات خطا |
| Application | Orchestration، Policyها، جلوگیری از وابستگی مستقیم UI به AD/EF/JWT |
| AD Adapter | اثبات Credential دستی، خواندن Snapshot و وضعیت حساب |
| Session/JWT | صدور، اعتبارسنجی، Refresh، Rotation، Reuse detection و Logout |
| SQL Server | Users، Sessions، Refresh Tokens، Audit و EF Migrations |
| Deployment Scripts | اعمال یا کنترل Desired State در App، IIS، AD، DNS و GPO |

### 5.3 ساختار Solution

    src/
      Portal.Domain/
      Portal.Application/
      Portal.Infrastructure/
      Portal.Web/
    tests/
      Portal.UnitTests/
      Portal.IntegrationTests/
      Portal.SecurityTests/
      Portal.E2ETests/
      Portal.LoadTests/
    deployment/
      parameters/
      scripts/
      iis/
      sql/
    docs/
      adr/
      runbooks/

- <code>Portal.Domain</code>: User، Session، Value Objectها و Policyهای خالص؛
- <code>Portal.Application</code>: Use Caseها و Interfaceها؛
- <code>Portal.Infrastructure</code>: EF Core، Active Directory، JWT، Certificate و Clock؛
- <code>Portal.Web</code>: Blazor SSR، endpointها، middleware و Composition Root.

برای جلوگیری از پیچیدگی غیرضروری، CQRS framework، Event Bus و Microservice در فاز اول اضافه نمی‌شوند.

---

## 6. جریان‌های اجرایی

### 6.1 ورود خودکار SSO

1. کاربر FQDN رسمی پرتال را باز می‌کند.
2. سامانه Access JWT و Refresh Session را بررسی می‌کند.
3. اگر نشست معتبر باشد، صفحه محافظت‌شده نمایش داده می‌شود.
4. اگر نشست وجود نداشته و Cookie توقف SSO موجود نباشد، سامانه یک SSO attempt دارای state/nonce آغاز می‌کند.
5. فقط endpoint مخصوص Windows SSO از IIS درخواست Kerberos Identity می‌کند.
6. IIS/Chrome با SPN رسمی Kerberos را انجام می‌دهند.
7. سامانه هویت <code>DOMAIN\username</code> و SID را به حساب همان دامین نگاشت می‌کند.
8. Snapshot حساب از AD خوانده و Status Gate اجرا می‌شود.
9. پروفایل محلی بر اساس <code>objectGUID</code> به‌صورت Idempotent ایجاد یا به‌روزرسانی می‌شود.
10. Session، Refresh Token و Audit موفقیت در یک Transaction ایجاد می‌شوند؛ شکست آن Transaction مانع صدور Cookie است.
11. Cookieها پیش از شروع Response نوشته و پاسخ با 303 به صفحه اصلی Redirect می‌شود.

### 6.2 ورود دستی

1. کاربر صفحه Login را باز می‌کند.
2. فرم فقط <code>sAMAccountName</code> ساده و Password را می‌پذیرد.
3. Anti-forgery، محدودیت طول، Origin و Rate Limit پیش از تماس با AD بررسی می‌شوند.
4. Username از دو طرف Trim می‌شود؛ مقدار خالی، بیش‌ازحد بلند، دارای <code>\</code>، <code>@</code>، NUL یا نویسه Control/Format رد می‌شود. مقدار ارسال‌شده به AD تغییر Case یا Unicode normalization نمی‌گیرد.
5. برای HMAC و Rate Limit، کلید canonical جداگانه با Unicode Form KC و UpperInvariant از Username ساخته می‌شود تا شکل‌های معادل محدودیت را دور نزنند.
6. Password باید غیرخالی و در سقف طول Configurable باشد و هیچ Trim، Normalize یا تغییر Case نمی‌گیرد.
7. Cooldown پایدار Username و limiter سریع IP به‌صورت Atomic بررسی می‌شوند.
8. <code>PrincipalContext.ValidateCredentials</code> دقیقاً یک بار و بدون Retry خودکار با <code>Negotiate | Signing | Sealing</code> اجرا می‌شود.
9. <code>false</code> فقط شکست Credential محسوب می‌شود؛ خطای DNS/DC/DirectoryServices به 503 نگاشت می‌شود و نباید به‌عنوان Password اشتباه Retry شود.
10. نام Writable DC متصل از Context دریافت می‌شود و Snapshot همان Login با هویت read-only gMSA از همان DC خوانده می‌شود. شکست آن DC پس از اثبات Password باعث Fail Closed و عدم سوییچ خودکار به DC دیگر در همان Login می‌شود.
11. Status Gate اجرا می‌شود.
12. همان مسیر مشترک ساخت User، Session، Refresh و JWT فراخوانی می‌شود و موفقیت شمارنده Username را Reset می‌کند.
13. پاسخ بیرونی خطا برای نام کاربری اشتباه، رمز اشتباه و حساب غیرمجاز یکسان است.

### 6.3 Logout و تعویض حساب

1. Logout فقط با POST و Anti-forgery مجاز است و می‌تواند نشست را از Access JWT معتبر یا Refresh credential معتبر شناسایی کند.
2. نشست <code>sid</code> و تمام Refresh Tokenهای همان نشست در SQL به‌صورت Idempotent باطل می‌شوند.
3. Access، Refresh و Anti-forgery Cookie حذف می‌شوند.
4. Cookie نشست‌محور <code>__Host-Portal-NoSso</code> ایجاد می‌شود.
5. کاربر با 303 به صفحه ورود دستی می‌رود.
6. ورود دستی با کاربر دیگر Windows Session را تغییر نمی‌دهد.
7. دکمه «ورود با حساب ویندوز» با یک POST محافظت‌شده، NoSso را حذف و SSO را آغاز می‌کند.
8. با پایان واقعی نشست Chrome، NoSso معمولاً حذف می‌شود و SSO خودکار برمی‌گردد.

محدودیت شناخته‌شده: Chrome ممکن است در برخی سیاست‌های Restore/Background، Session Cookie را بازیابی کند. آزمون E2E روی Chrome سازمانی الزامی است و دکمه ورود با حساب ویندوز همیشه مسیر بازیابی قطعی خواهد بود.

### 6.4 Refresh

1. Refresh فقط با POST، Anti-forgery و Refresh Cookie معتبر انجام می‌شود.
2. Hash توکن، نشست، Idle Expiry، Absolute Expiry و Revocation در Transaction بررسی می‌شوند.
3. وضعیت حساب AD دوباره کنترل می‌شود.
4. Refresh Token قبلی Consumed و توکن جدید ایجاد می‌شود.
5. JWT جدید با همان <code>sid</code> و <code>jti</code> جدید صادر می‌شود.
6. درخواست هم‌زمان با همان توکن در Grace پنج‌ثانیه‌ای، پاسخ <code>409 Conflict</code> و <code>Retry-After: 1</code> بدون JWT، Refresh Token یا Set-Cookie جدید می‌گیرد و نشست را باطل نمی‌کند؛ reuse خارج از Grace کل نشست را باطل می‌کند.
7. برای فرم‌های طولانی، یک ماژول کوچک JavaScript فقط Refresh را با Web Lock بین تب‌های Chrome هماهنگ می‌کند؛ هیچ Token برای JavaScript قابل خواندن نیست.

اگر پاسخ درخواست برنده پیش از Set-Cookie از دست برود، بازیابی توکن خام ممکن نیست؛ سیستم Fail Closed می‌کند و Re-login لازم است. توکن جایگزین خام فقط برای جبران race در SQL ذخیره یا قابل بازیابی نمی‌شود.

### 6.5 رفتار خطا

| وضعیت | رفتار |
|---|---|
| Credential یا حساب نامعتبر | پیام عمومی یکسان؛ بدون Token |
| Rate Limit | HTTP 429 و Retry-After |
| AD در دسترس نیست | HTTP 503 و Correlation ID؛ بدون Login/Refresh |
| SQL در دسترس نیست | Fail Closed برای تمام مسیرهای محافظت‌شده |
| Kerberos ناموفق | ثبت نتیجه فنی و راهنمایی best-effort به مسیر دستی؛ Chrome ممکن است پیش از بازگشت کنترل به برنامه Credential Prompt یا 401 بومی نمایش دهد. فرم دستی همیشه با URL مستقیم در دسترس است |
| JWT نامعتبر/منقضی | تلاش Refresh کنترل‌شده یا بازگشت به Login |
| Session باطل/منقضی | حذف Cookieهای محلی و Login |
| خطای داخلی | صفحه عمومی بدون Stack Trace یا جزئیات زیرساخت |

---

## 7. طراحی امنیت

### 7.1 IIS، Kerberos و Chrome

- FQDN ثابت و HTTPS تنها مسیر رسمی است؛ DNS به‌صورت A Record مستقیم تعریف می‌شود و استفاده از IP یا CNAME تأییدنشده پشتیبانی نمی‌شود.
- Windows Authentication Role Service و .NET 10 Hosting Bundle نصب می‌شوند.
- App Pool اختصاصی، x64، No Managed Code و In-Process است.
- App Pool روی <code>startMode=AlwaysRunning</code>، Site/Application روی <code>preloadEnabled=true</code> و Idle Timeout روی صفر قرار می‌گیرد تا cleanup داخلی و cold start قابل اتکا باشند؛ این تنظیمات در Integration آزموده می‌شوند.
- App Pool با gMSA اختصاصی و بدون رمز ذخیره‌شده اجرا می‌شود.
- SPN یکتای <code>HTTP/portal.mydomain.ir</code> روی همان gMSA ثبت می‌شود.
- <code>useKernelMode=true</code> و <code>useAppPoolCredentials=true</code> پس از آزمون تنظیم می‌شوند.
- Windows Authentication provider در IIS فقط Negotiate است و پذیرش منفی NTLM با Windows Security Policy روی سرور، پس از Impact Assessment، enforce می‌شود.
- Anonymous و Windows Authentication در IIS به‌گونه‌ای تنظیم می‌شوند که Application بتواند فقط برای policy اختصاصی SSO Challenge ایجاد کند.
- <code>AutomaticAuthentication=false</code> و scheme پیش‌فرض برنامه JWT است.
- فقط policy مسیر SSO Windows scheme را می‌پذیرد؛ داشتن JWT نباید آن policy را ارضا کند.
- Chrome GPO شامل FQDN دقیق در <code>AuthServerAllowlist</code> است؛ wildcard استفاده نمی‌شود.
- Delegation لازم نیست و gMSA روی Trusted for Delegation قرار نمی‌گیرد.
- Kerberos با <code>klist</code> و Windows Security Log یا Capture اثبات می‌شود؛ عنوان Negotiate به‌تنهایی کافی نیست.
- یک کلاینت آزمایشی که عمداً NTLM را پیشنهاد می‌دهد باید رد شود. تا زمانی که این تست منفی پاس نشده است، برنامه مجاز به صدور <code>amr=["kerberos"]</code> در Production نیست.
- NTLM در Production معیار پذیرش نیست. مسدودسازی واقعی NTLM باید پس از Impact Assessment روی همان سرور و محیط Integration اجرا شود تا سرویس دیگری آسیب نبیند.
- اگر روی سرور مشترک امکان جلوگیری قابل اثبات از NTLM وجود نداشته باشد، Go-Live با شرط Kerberos-only متوقف می‌شود و ادامه فقط با Change Request صریح ممکن است.
- شرط Kerberos-only به مسیر SSO ورودی از Chrome مربوط است. ورود دستی از SSPI Negotiate با Signing/Sealing استفاده می‌کند و ممکن است در ارتباط Backend با DC از مکانیزم دیگری استفاده کند؛ این مسیر با <code>amr=["pwd"]</code> شناخته می‌شود و SSO محسوب نمی‌شود.
- Extended Protection ابتدا در Integration روی Allow و پس از اثبات سازگاری، در صورت امکان روی Require قرار می‌گیرد.

### 7.2 ورود دستی و Active Directory

Interfaceهای اصلی:

| Interface | مسئولیت |
|---|---|
| <code>IManualAuthenticationService</code> | هماهنگ‌سازی جریان ورود |
| <code>IAdCredentialValidator</code> | اثبات Password بدون ذخیره آن |
| <code>IAdUserDirectory</code> | خواندن Snapshot حساب |
| <code>IAdAccountEligibilityPolicy</code> | Status Gate خالص و قابل تست |
| <code>ILoginAttemptGuard</code> | Rate Limit و Cooldown |
| <code>ILocalUserSynchronizer</code> | Upsert بر اساس شناسه پایدار AD |
| <code>ISessionIssuer</code> | صدور مشترک نشست برای SSO و Manual |
| <code>ISecurityAuditWriter</code> | Audit پاک‌سازی‌شده |

Status Gate فقط در صورت برقراری همه شروط اجازه صدور نشست می‌دهد:

- شیء AD وجود داشته و User باشد؛
- <code>Enabled == true</code>؛
- <code>IsAccountLockedOut() == false</code>؛
- AccountExpirationDate خالی یا بعد از زمان جاری UTC باشد؛
- اگر <code>pwdLastSet == 0</code> باشد، نتیجه PasswordChangeRequired است؛
- اگر <code>DONT_EXPIRE_PASSWD</code> یا مقدار ویژه <code>0x7FFFFFFFFFFFFFFF</code> برقرار باشد، رمز Never Expires تلقی می‌شود و مقدار ویژه مستقیماً به DateTime تبدیل نمی‌شود؛
- در غیر این صورت، فعال‌بودن <code>UF_PASSWORD_EXPIRED</code>، مقدار صفر نامعتبر یا FILETIME گذشته در <code>msDS-UserPasswordExpiryTimeComputed</code> نتیجه PasswordExpired می‌دهد؛
- <code>objectGUID</code> و SID قابل بازیابی باشند.

هیچ LDAP Simple Bind روی پورت 389 انجام نمی‌شود. Password در کمترین Scope ممکن نگهداری شده و وارد Component State، Cache، SQL، Claims، Audit یا Request-body logging نمی‌شود.

عملیات PrincipalContext همگام است و Cancellation قطعی برای call در حال اجرا ندارد. کنترل اجرایی آن شامل Bulkhead محدود، زمان انتظار ورود به Bulkhead حداکثر ۲۵۰ میلی‌ثانیه، Queue صفر، Circuit Breaker فقط برای خطاهای زیرساختی و یک آزمون Blackhole DC است. مقدار اولیه هم‌زمانی ۲۰ است و فقط پس از Load Test افزایش می‌یابد. اگر callهای گیرکرده از بودجه زمانی پذیرش عبور کنند، Adapter پیش از Go-Live باید به پیاده‌سازی دارای Timeout واقعی مانند LdapConnection تغییر کند؛ Task.Run به‌عنوان Timeout کاذب پذیرفته نیست.

### 7.3 Rate Limit

مقادیر مصوب اولیه:

- ۵ شکست برای هر نام کاربری در ۱۵ دقیقه؛
- ۲۰ شکست برای هر IP در ۱۵ دقیقه؛
- Cooldown برابر ۱۵ دقیقه؛
- تأخیر افزایشی و jitter کوتاه پس از شکست؛
- حداکثر عملیات هم‌زمان AD به‌صورت Configurable، مقدار اولیه ۲۰؛
- Queue عملیات حاوی Password صفر یا بسیار کوتاه؛
- شمارنده نام کاربری با HMAC نگهداری می‌شود، نه نام خام؛
- Lockout دائمی مستقل در SQL ساخته نمی‌شود.
- محدودیت سریع IP با Rate Limiter داخلی ASP.NET Core و وضعیت Cooldown نام کاربری به‌صورت پایدار با HMAC در SQL نگهداری می‌شود؛ Recycle IIS نباید Cooldown کاربر را دور بزند.

قبل از Production، این مقادیر باید کمتر از Lockout Threshold واقعی AD تنظیم شوند. هر Submit باید حداکثر یک تلاش مؤثر Credential در AD ایجاد کند.

الگوریتم:

1. IP limiter همه درخواست‌های Login را پیش از Model Binding سنگین شمارش می‌کند.
2. State نام کاربری در SQL با HMAC canonical، Transaction و RowVersion خوانده می‌شود؛ BlockedUntil معتبر فوراً 429 می‌دهد.
3. فقط نتیجه‌های کاربری مانند Invalid Credential یا حساب غیرمجاز شمارنده شکست را Atomic افزایش می‌دهند؛ خطای AD/SQL زیرساختی شمارنده کاربر را افزایش نمی‌دهد.
4. رسیدن به آستانه، BlockedUntil را روی Cooldown کوتاه تنظیم می‌کند.
5. Login موفق State نام کاربری را Reset می‌کند.
6. Recycle فقط limiter حافظه‌ای IP را Reset می‌کند؛ Cooldown نام کاربری باقی می‌ماند.
7. چون IIS مستقیم و بدون Proxy/Load Balancer است، Remote IP همان Client IP فرض می‌شود. اگر Proxy یا NAT مشترک اضافه شود، این فرض و آستانه IP باید با Change Request بازطراحی شوند.
8. کلید HMAC نام کاربری، IP/Audit و Refresh از نظر Purpose و Environment جدا هستند.

### 7.4 JWT

Header:

- <code>alg = RS256</code>
- <code>typ = at+jwt</code>
- <code>kid = شناسه گواهی امضاکننده</code>

Claims حداقلی:

| Claim | مقدار |
|---|---|
| <code>iss</code> | Issuer تنظیم‌شده |
| <code>aud</code> | <code>portal-web</code> |
| <code>client_id</code> | <code>portal-web</code> |
| <code>sub</code> | GUID کاربر محلی |
| <code>ad_object_guid</code> | شناسه پایدار AD |
| <code>sid</code> | شناسه نشست SQL |
| <code>jti</code> | شناسه یکتای JWT |
| <code>amr</code> | JSON array ثابت: <code>["kerberos"]</code> برای SSO یا <code>["pwd"]</code> برای فرم دستی |
| <code>auth_time</code> | زمان اثبات هویت |
| <code>iat/nbf/exp</code> | زمان‌های استاندارد UTC |
| <code>name</code> | فقط برای نمایش، در صورت نیاز |

Password، Refresh Token، IP، وضعیت Lockout، Groupها و Roleهای AD داخل JWT قرار نمی‌گیرند. JWT امضاشده رمزگذاری‌شده نیست.

این Token یک access token داخلی و اختصاصی پرتال است و سامانه در فاز اول OAuth Authorization Server عمومی محسوب نمی‌شود؛ استفاده از <code>at+jwt</code> و Claims بالا برای تفکیک نوع و هم‌راستایی امنیتی است، نه ادعای ارائه کامل OAuth.

Validation الزامی:

- امضا، الگوریتم ثابت RS256 و <code>kid</code>؛
- issuer، audience، type، lifetime و ClockSkew صریح؛
- <code>client_id=portal-web</code> و واژگان ثابت amr؛
- وجود Claims اجباری؛
- تطابق <code>sub</code> با مالک Session؛
- فعال‌بودن <code>sid</code> در SQL؛
- عدم پذیرش Authorization Header در scheme مخصوص مرورگر؛ JWT فقط از Cookie خوانده می‌شود؛
- Fail Closed در Timeout یا قطعی SQL.

### 7.5 Cookie و CSRF

| Cookie | محتوا | تنظیم |
|---|---|---|
| <code>__Host-Portal-Access</code> | JWT | Secure، HttpOnly، SameSite=Strict، Path=/، بدون Domain |
| <code>__Host-Portal-Refresh</code> | مقدار تصادفی opaque | همان سیاست؛ Session Cookie |
| <code>__Host-Portal-NoSso</code> | توقف SSO خودکار | همان سیاست؛ Session Cookie |
| <code>__Host-Portal-Antiforgery</code> | Cookie ضد CSRF | Secure، HttpOnly، SameSite=Strict، Path=/، بدون Domain؛ Request Token جداگانه داخل فرم/هدر |

- Login، Windows-login، Refresh و Logout با Anti-forgery محافظت می‌شوند.
- یک scheme محدود به مسیرهای Refresh و Logout، Refresh credential و Session را پیش از Anti-forgery اعتبارسنجی می‌کند و Principal دارای همان sub/sid می‌سازد؛ بنابراین Access JWT منقضی مانع اعتبارسنجی Anti-forgery نمی‌شود.
- عملیات تغییر وضعیت با GET انجام نمی‌شود. Windows callback یک استثنای صریح پروتکلی است و فقط state محافظت‌شده، browser-bound، دو دقیقه‌ای و single-use را می‌پذیرد.
- state دارای شناسه تصادفی، return URL محلی، زمان صدور و browser binding است؛ Hash آن در SsoChallenges به‌صورت Atomic مصرف می‌شود و replay یا callback مستقیم رد می‌گردد.
- Origin برای endpointهای Authentication با Host رسمی تطبیق داده می‌شود.
- CORS غیرفعال است.
- Redirect فقط به URL محلی و اعتبارسنجی‌شده مجاز است.
- صفحات Auth دارای <code>Cache-Control: no-store</code> و بدون Streaming هستند.
- POST/PUT/PATCH/DELETE منقضی هرگز خودکار Redirect یا Replay نمی‌شوند، زیرا بدنه ممکن است از بین برود. در فاز اول فقط endpointهای Auth وجود دارند و هرکدام قرارداد Refresh/Logout مستقل دارند؛ Featureهای تجاری آینده باید Draft/Retry را جدا طراحی کنند.

### 7.6 گواهی و Data Protection

- گواهی JWT از گواهی TLS جداست.
- گواهی RSA اختصاصی در <code>LocalMachine\My</code> نگهداری می‌شود.
- Private Key فقط برای gMSA قابل خواندن و ترجیحاً Non-exportable است.
- فایل تنظیمات فقط Thumbprint و KeyId را نگه می‌دارد.
- یک Machine Secret Provider واحد، کلیدهای تصادفی ۲۵۶بیتی HMAC را به‌صورت DPAPI-protected در Registry محیط و با ACL محدود به gMSA نگهداری می‌کند.
- کلیدهای HMAC برای Refresh، Username limiter، IP و User-Agent از نظر Purpose و Environment جدا هستند؛ هر کلید دارای HashKeyId، ActivatedAt و RetireAfter است.
- کلید قبلی Refresh حداقل تا پایان AbsoluteSessionLifetime به‌علاوه پنجره Rollback نگهداری می‌شود؛ کلید Rate Limit تا پایان Window+Cooldown و کلید Audit تا پایان Retention لازم برای هم‌بستگی حفظ می‌شود.
- برنامه در نبود گواهی، Private Key یا انقضای آن Fail Fast می‌کند.
- برای Rotation، گواهی جدید ابتدا به مجموعه Validation اضافه، سپس Signing Key می‌شود. Private Key قبلی تا پایان پنجره Rollback نگهداری می‌شود؛ حذف آن صرفاً پس از Access lifetime مجاز نیست.
- Data Protection مستقل از JWT و برای Anti-forgery و state لازم است.
- Key Ring هر محیط روی Registry path مستقل مانند <code>HKLM\SOFTWARE\MyOrg\Portal\Production\DataProtection</code> نگهداری، با DPAPI ماشین محافظت و فقط برای gMSA و Administrators قابل دسترسی است.
- <code>SetApplicationName</code> برای Production برابر <code>MyOrg.Portal.Production</code> و برای Integration مقدار متفاوت دارد؛ Cookie و state دو محیط قابل تبادل نیستند.
- Recycle IIS نباید کلیدهای Data Protection را از بین ببرد.

---

## 8. مدل داده

### 8.1 جدول Users

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| Id | uniqueidentifier PK | شناسه داخلی و مقدار <code>sub</code> |
| AdObjectGuid | uniqueidentifier | شناسه پایدار AD؛ Unique |
| AdSid | nvarchar(184) | SID؛ Unique |
| SamAccountName | nvarchar(256) | نام کاربری جاری |
| NormalizedSamAccountName | nvarchar(256) | Index غیرUnique برای جست‌وجو |
| UserPrincipalName | nvarchar(1024) null | Snapshot مطابق سقف محافظه‌کارانه AD |
| DisplayName | nvarchar(256) null | نمایش UI |
| FirstSeenAtUtc | datetime2(3) | اولین ورود |
| LastLoginAtUtc | datetime2(3) | آخرین ورود موفق |
| LastDirectoryValidationAtUtc | datetime2(3) | آخرین Status Gate |
| CreatedAtUtc / UpdatedAtUtc | datetime2(3) | Audit فنی |
| RowVersion | rowversion | کنترل هم‌زمانی |

Enabled و Lockout در صورت ذخیره فقط Snapshot اطلاعاتی هستند و مرجع نهایی مجوز محسوب نمی‌شوند. تغییر SamAccountName باید همان رکورد را بر اساس AdObjectGuid به‌روزرسانی کند. NormalizedSamAccountName عمداً Unique نیست، زیرا AD می‌تواند یک نام قدیمی را پس از Rename به objectGUID دیگری واگذار کند؛ Resolver ابتدا کاربر جاری را از AD می‌یابد و سپس با AdObjectGuid در SQL هم‌بسته می‌کند.

### 8.2 جدول AuthSessions

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| Sid | uniqueidentifier PK | شناسه نشست |
| UserId | uniqueidentifier FK | مالک نشست |
| AuthenticationMethod | tinyint | Kerberos یا Password |
| CreatedAtUtc | datetime2(3) | زمان ایجاد |
| LastActivityAtUtc | datetime2(3) | آخرین Refresh معتبر |
| IdleExpiresAtUtc | datetime2(3) | انقضای بی‌کاری |
| AbsoluteExpiresAtUtc | datetime2(3) | سقف نهایی |
| RevokedAtUtc | datetime2(3) null | زمان ابطال |
| RevocationReason | nvarchar(64) null | Logout، reuse، account status و غیره |
| CreatedIpKeyHmac | binary(32) null | HMAC purpose-specific، نه Hash ساده |
| IpHashKeyId | smallint null | نسخه کلید IP |
| UserAgentKeyHmac | binary(32) null | HMAC جداگانه برای User-Agent |
| UserAgentHashKeyId | smallint null | نسخه کلید User-Agent |
| RowVersion | rowversion | کنترل Refresh هم‌زمان |

Indexها:

- Primary Key روی Sid؛
- Index روی UserId؛
- Filtered Index برای نشست‌های فعال؛
- Index روی IdleExpiresAtUtc و AbsoluteExpiresAtUtc برای Cleanup.

### 8.3 جدول RefreshTokens

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| Id | uniqueidentifier PK | شناسه توکن |
| Sid | uniqueidentifier FK | خانواده نشست |
| TokenHash | binary(32) | HMAC-SHA256؛ Unique |
| HashKeyId | smallint | نسخه کلید HMAC |
| Generation | int | نسل Rotation |
| CreatedAtUtc / ExpiresAtUtc | datetime2(3) | چرخه عمر |
| ConsumedAtUtc | datetime2(3) null | مصرف موفق |
| ReplacedById | uniqueidentifier null | توکن بعدی |
| RevokedAtUtc | datetime2(3) null | ابطال |
| ReuseDetectedAtUtc | datetime2(3) null | تشخیص replay |
| RowVersion | rowversion | هم‌زمانی |

Refresh Token از ۳۲ بایت تصادفی امن ساخته می‌شود. فقط HMAC آن با کلید نسخه‌دار خارج از دیتابیس ذخیره می‌شود. Token خام هرگز در SQL یا Audit نوشته نمی‌شود.

### 8.4 جدول AuthenticationAuditEvents

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| Id | bigint identity PK | شناسه رخداد |
| OccurredAtUtc | datetime2(3) | زمان UTC |
| EventType | nvarchar(64) | Login، Refresh، Logout، Revoke و غیره |
| Outcome | tinyint | Success، Failure، Throttled، InfrastructureError |
| FailureCode | nvarchar(64) null | Reason داخلی پاک‌سازی‌شده |
| UserId | uniqueidentifier null | فقط در صورت Resolve شدن |
| Sid | uniqueidentifier null | نشست مرتبط |
| AuthenticationMethod | tinyint null | SSO یا Manual |
| CorrelationId | nvarchar(64) | ردیابی |
| UsernameKeyHmac | binary(32) null | شکست‌ها بدون ذخیره Username خام |
| UsernameHashKeyId | smallint null | نسخه کلید Username |
| IpKeyHmac / UserAgentKeyHmac | binary(32) null | HMAC purpose-specific؛ نه Hash قابل حدس |
| PseudonymHashKeyId | smallint null | نسخه کلید Audit |

Retention پیش‌فرض ۱۸۰ روز است. Cleanup داده‌های منقضی یک وظیفه داخلی زمان‌بندی‌شده است و Monitoring محسوب نمی‌شود.

رویداد موفق Login، Refresh و Logout باید تا حد ممکن در همان Transaction تغییر Session ثبت شود. در صورت ناموفق‌بودن ثبت Audit موفقیت، Credential یا Cookie جدید صادر نمی‌شود.

### 8.5 جدول LoginAttemptStates

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| KeyType | tinyint | Username یا کلید پایدار دیگر |
| KeyHmac | binary(32) | HMAC نام Normalized؛ بخشی از PK |
| HashKeyId | smallint | نسخه کلید HMAC |
| WindowStartedAtUtc | datetime2(3) | شروع پنجره |
| FailedCount | int | شمارنده محدود |
| BlockedUntilUtc | datetime2(3) null | Cooldown موقت |
| LastAttemptAtUtc | datetime2(3) | Cleanup و Audit |
| RowVersion | rowversion | کنترل هم‌زمانی |

این جدول Lockout دائمی مستقل ایجاد نمی‌کند؛ فقط Cooldown کوتاه و مصوب را پس از Recycle IIS حفظ می‌کند. Rate Limit سریع IP در حافظه برنامه باقی می‌ماند و AD مرجع نهایی Lockout است.

### 8.6 جدول SsoChallenges

| ستون | نوع پیشنهادی | توضیح |
|---|---|---|
| Id | uniqueidentifier PK | شناسه challenge |
| StateHash | binary(32) | HMAC state؛ Unique |
| BrowserBindingHmac | binary(32) | اتصال به مرورگر آغازگر |
| LocalReturnUrl | nvarchar(1024) | فقط مسیر محلی |
| CreatedAtUtc / ExpiresAtUtc | datetime2(3) | TTL پیش‌فرض دو دقیقه |
| ConsumedAtUtc | datetime2(3) null | مصرف یک‌باره |
| RowVersion | rowversion | جلوگیری از replay هم‌زمان |

Callback مستقیم، state منقضی، state مصرف‌شده و browser binding نامعتبر رد می‌شوند. مقدار state خام فقط در Cookie امن کوتاه‌عمر و payload محافظت‌شده Data Protection وجود دارد.

### 8.7 اصول EF Core و Migration

- EF Core هم‌نسخه با .NET 10 و SQL Server provider رسمی استفاده می‌شود.
- DbContext کوتاه‌عمر و No-Tracking برای lookup نشست در نظر گرفته می‌شود.
- تمام DateTimeها UTC و ترجیحاً از TimeProvider تزریق‌شده دریافت می‌شوند.
- Constraint، Unique Index، Foreign Key و RowVersion در خود Schema اعمال می‌شوند.
- Migration در Startup برنامه خودکار اجرا نمی‌شود.
- Migration Bundle یا اسکریپت idempotent در Change Window اجرا می‌شود.
- چون Backup خارج از محدوده است، Migration فاز اول فقط Expand-only و غیرمخرب است.
- حذف یا Rename مخرب ستون، تبدیل برگشت‌ناپذیر داده و هر Migration ناسازگار با نسخه قبلی ممنوع است.
- نسخه قبلی برنامه باید در بازه Rollback با Schema جدید سازگار بماند.

---

## 9. فایل مرجع تنظیمات

### 9.1 اصل Source of Truth

یک فایل غیرمحرمانه برای هر محیط، مانند <code>portal.parameters.Production.json</code>، Desired State را تعریف می‌کند. برنامه و اسکریپت‌های PowerShell همان فایل را می‌خوانند.

این فایل نمی‌تواند محل واقعی تمام تنظیمات باشد؛ SPN در AD، Authentication در IIS و Chrome Policy در GPO ذخیره می‌شوند. اسکریپت Preflight اختلاف Desired State با وضعیت واقعی را گزارش و در صورت مجازبودن اعمال می‌کند.

### 9.2 بخش‌های فایل

    {
      "SchemaVersion": 1,
      "Application": {
        "Name": "Portal",
        "Environment": "Production",
        "BaseUrl": "https://portal.mydomain.ir",
        "DisplayTimeZone": "Asia/Tehran"
      },
      "ActiveDirectory": {
        "DnsDomain": "corp.mydomain.ir",
        "NetBiosName": "MYDOMAIN",
        "RequireKerberosForSso": true,
        "CredentialValidation": "NegotiateSigningSealing",
        "StatusRevalidationMinutes": 10,
        "MaxConcurrentOperations": 20,
        "BulkheadWaitMilliseconds": 250,
        "InfrastructureFailureThreshold": 5,
        "CircuitBreakSeconds": 30,
        "MaxUsernameLength": 256,
        "MaxPasswordLength": 256
      },
      "Authentication": {
        "AutoSsoOnNewBrowserSession": true,
        "AccessTokenSeconds": 570,
        "IdleTimeoutMinutes": 30,
        "AbsoluteSessionHours": 8,
        "RefreshConcurrencyGraceSeconds": 5
      },
      "Jwt": {
        "Issuer": "https://portal.mydomain.ir",
        "Audience": "portal-web",
        "Algorithm": "RS256",
        "ClockSkewSeconds": 30,
        "CurrentCertificateThumbprint": "<PLACEHOLDER>",
        "PreviousCertificateThumbprints": []
      },
      "Cryptography": {
        "CurrentRefreshHmacKeyId": 1,
        "CurrentRateLimitHmacKeyId": 1,
        "CurrentAuditHmacKeyId": 1,
        "DataProtectionApplicationName": "MyOrg.Portal.Production",
        "DataProtectionRegistryPath": "HKLM\\SOFTWARE\\MyOrg\\Portal\\Production\\DataProtection"
      },
      "Database": {
        "ConnectionString": "Server=<sql-fqdn>;Database=PortalAuth;Integrated Security=True;Encrypt=True;TrustServerCertificate=False"
      },
      "RateLimiting": {
        "UserFailures": 5,
        "UserWindowMinutes": 15,
        "IpFailures": 20,
        "IpWindowMinutes": 15,
        "CooldownMinutes": 15
      },
      "Audit": {
        "RetentionDays": 180
      },
      "Infrastructure": {
        "IisSiteName": "Portal",
        "AppPoolName": "Portal",
        "Gmsa": "MYDOMAIN\\gmsaPrtlProd$",
        "Spn": "HTTP/portal.mydomain.ir",
        "ChromeAuthServerAllowlist": "portal.mydomain.ir"
      }
    }

### 9.3 قواعد

- JSON Schema و Strongly Typed Options برای تمام بخش‌ها تعریف می‌شوند.
- <code>ValidateOnStart</code> ناسازگاری، مقدار خارج از محدوده، Thumbprint ناقص و URL نامعتبر را Fail Fast می‌کند.
- Secret، Password، Private Key و Refresh HMAC Key داخل فایل یا Git قرار نمی‌گیرند.
- کلیدهای رمزنگاری از Certificate Store، DPAPI یا Environment محافظت‌شده IIS دریافت می‌شوند.
- Connection String به‌دلیل Integrated Security فاقد Password است.
- تغییر فایل Production مستلزم Change Record و Recycle کنترل‌شده App Pool است.
- تمام فایل‌های محیط دارای SchemaVersion هستند تا اسکریپت قدیمی فایل جدید را اشتباه تفسیر نکند.
- اسکریپت Drift Check مقادیر IIS، SPN، gMSA، DNS، Certificate و Chrome Policy را با فایل مقایسه می‌کند.

---

## 10. Endpointها و صفحات

| مسیر | روش | دسترسی | مسئولیت |
|---|---|---|---|
| <code>/account/login</code> | GET | Anonymous | Static SSR؛ فرم دستی و دکمه Windows |
| <code>/account/renew</code> | GET | Refresh scheme | Static SSR بدون Streaming؛ تولید Request Token جدید و POST کنترل‌شده Refresh |
| <code>/auth/manual-login</code> | POST | Anonymous + Anti-forgery + Rate Limit | ورود دستی AD |
| <code>/auth/windows-login</code> | POST | Anonymous + Anti-forgery | حذف NoSso و آغاز SSO |
| <code>/auth/windows/callback</code> | GET/Callback | Windows policy + state/nonce | Exchange هویت Windows با Session |
| <code>/auth/refresh</code> | POST | Refresh scheme + Anti-forgery | Rotation و JWT جدید |
| <code>/auth/logout</code> | POST | Access یا Refresh credential + Anti-forgery | ابطال Idempotent نشست جاری، حتی اگر Access JWT منقضی شده باشد |
| <code>/auth/me</code> | GET | JWT | اطلاعات حداقلی کاربر/نشست |
| <code>/</code> | GET | JWT | صفحه اصلی محافظت‌شده |
| <code>/errors/unauthorized</code> | GET | Anonymous | خطای دسترسی |
| <code>/errors/throttled</code> | GET | Anonymous | خطای Rate Limit |
| <code>/errors/unavailable</code> | GET | Anonymous | خطای AD/SQL با Correlation ID |
| <code>/errors/general</code> | GET | Anonymous | خطای عمومی |

قواعد:

- Fallback Authorization Policy تمام endpointها را به‌طور پیش‌فرض محافظت می‌کند.
- فقط مسیرهای صریح Login/Error/Refresh و SSO initiation ناشناس هستند.
- Access token منقضی ممکن است باعث ورود کنترل‌شده به Refresh شود؛ Refresh بدون Credential معتبر هیچ اثری ندارد.
- Endpoint تشخیصی نمایش Windows Principal در Production وجود ندارد.
- Authentication cookieها فقط پیش از شروع Response در endpoint یا middleware نوشته می‌شوند.
- یک Authorization result handler میان HTML و endpoint داده‌ای تفکیک می‌کند: GET/HEAD دارای Accept برابر text/html فقط به renew یا login با 303 و returnUrl محلی می‌روند؛ <code>/auth/me</code> و درخواست غیرHTML همان 401/403 را برمی‌گردانند؛ روش‌های unsafe هرگز Redirect نمی‌شوند.
- guard حلقه Redirect مانع برگشت Login به خودش یا زنجیره renew نامحدود می‌شود.

### 10.1 رابط کاربری

- فارسی و RTL؛
- صفحه Login شامل نام ثابت دامین، Username، Password، ورود دستی و ورود با Windows؛
- عدم وجود Remember Me، Sign Up، Forgot Password یا Password Change؛
- صفحه اصلی شامل Display Name، Username، روش ورود، زمان ورود و Logout؛
- Errorها بدون اطلاعات فنی و همراه Correlation ID؛
- Semantic HTML، Focus قابل مشاهده، Keyboard navigation و Label صحیح؛
- پشتیبانی رسمی فقط از آخرین نسخه سازمانی Chrome روی Windows.

---

## 11. پیش‌نیازهای فاز صفر

### 11.1 ورودی‌های الزامی

- AD DNS Domain و NetBIOS واقعی؛
- Production و Integration FQDN؛
- نسخه و Patch Level واقعی Windows Server، IIS، Chrome و SQL Server؛
- مشخصات CPU، RAM و Storage سرور؛
- Lockout Policy و Fine-Grained Password Policyهای مؤثر؛
- نام gMSAها، OU پایلوت، حساب‌های تست و مسئول هر تغییر؛
- Thumbprint گواهی TLS وب، گواهی معتبر SQL و گواهی JWT؛
- Window مجاز تغییر و UAT؛
- نام SQL Instance و دسترسی gMSA؛
- تأیید اینکه مسیر Client تا IIS فاقد Proxy/NAT مشترک است یا اعلام توپولوژی واقعی Client IP؛
- تأیید کتبی ریسک نبود Backup و Monitoring.

### 11.2 Preflight

1. Domain Join و Secure Channel سرور و کلاینت پایلوت؛
2. A Record مستقیم DNS، resolution مستقیم و معکوس و همگامی ساعت؛ CNAME فقط با SPN/GPO/Chrome test جدا مجاز است؛
3. KDS Root Key و امکان نصب gMSA؛
4. عدم وجود SPN تکراری برای FQDN؛
5. Trust Chain و تطابق SAN گواهی TLS با FQDN؛
6. نصب IIS پیش از Hosting Bundle یا Repair آن؛
7. Windows Authentication Role Service؛
8. App Pool x64، In-Process و No Managed Code؛
9. دسترسی gMSA به SQL، پوشه برنامه، Data Protection و Private Key؛
10. پروتکل اتصال محلی SQL و اعتبار گواهی آن؛ فعال‌کردن <code>TrustServerCertificate=true</code> به‌صورت پنهانی مجاز نیست؛
11. Chrome ADMX/ADML در Central Store و قابلیت Machine Policy؛
12. نمایش <code>AuthServerAllowlist</code> با وضعیت OK در <code>chrome://policy</code>؛
13. امکان مسدودسازی NTLM بدون آسیب به سرویس‌های دیگر روی سرور؛
14. نبود Password یا Secret در فایل‌های پارامتر و Artifact؛
15. آماده‌بودن Site و DB جدا برای Integration؛
16. آماده‌بودن حساب‌های تست برای تمام وضعیت‌های AD.

### 11.3 محیط Integration حداقلی

- Site و App Pool جدا؛
- FQDN، Binding، Certificate، SPN و gMSA جدا؛
- Database و Migration History جدا؛
- Cookie prefix یا Host جدا برای جلوگیری از تداخل؛
- OU/GPO پایلوت و چند کلاینت محدود؛
- حساب‌های تست: Valid، Disabled، Locked، Account Expired، Password Expired، Must Change Password و Rename شده؛
- دسترسی فقط تیم پروژه؛
- داده ساختگی و بدون Credential کاربران واقعی؛
- تست بار فقط پیش از Production و در Change Window.
- پس از Go-Live، Integration App Pool به‌طور پیش‌فرض Stop/Disabled می‌ماند و فقط در Change Window روشن می‌شود تا با Production بر سر CPU/RAM/IO رقابت نکند.

---

## 12. کیفیت و استانداردهای کدنویسی

- Nullable Reference Types و Treat Warnings as Errors؛
- تحلیل‌گرهای رسمی .NET و Formatting ثابت؛
- Dependency Injection و Interface فقط در مرزهای دارای ارزش تست/تعویض؛
- عدم استفاده از Singleton برای PrincipalContext/UserPrincipal بدون اثبات Thread Safety؛
- Concurrency Bulkhead برای APIهای همگام DirectoryServices؛
- Cancellation و Timeout در مرزهای SQL و HTTP؛
- TimeProvider برای تمام منطق زمانی؛
- RandomNumberGenerator برای Tokenها؛
- CryptographicOperations برای مقایسه‌های حساس در صورت کاربرد؛
- عدم استفاده از الگوریتم قابل انتخاب توسط ورودی Token؛
- Parameterized query از طریق EF Core؛
- Redaction مرکزی Header، Cookie، Password و Token؛
- Dependency vulnerability scan و Lock کردن نسخه packageها؛
- ADR برای تصمیم‌های JWT-in-Cookie، Kerberos-only، Session lookup و Configuration planes؛
- Code Review اجباری برای Authentication، Crypto، Migration و Deployment scripts؛
- هیچ Feature تشخیصی ناامن با Environment flag قابل فعال‌شدن در Production نباشد.

---

## 13. برنامه آزمون

### 13.1 هرم آزمون

| سطح | هدف | محل اجرا |
|---|---|---|
| Unit | Policy، زمان، Claims، Parser، Rotation و Redaction | هر Build |
| Component/Integration | EF Core، SQL، endpoint و Fake AD | Development/Integration |
| Real AD Integration | Credential، Status، gMSA و خرابی DC | Integration |
| IIS/Chrome E2E | Kerberos، GPO، Cookie و تعویض حساب | Integration Domain |
| Security | CSRF، Replay، Enumeration و Hardening | Integration |
| Performance | ۱٬۰۰۰/۱٬۵۰۰ نشست و موج Login/Refresh | پیش از Go-Live |
| UAT | تجربه واقعی کاربر و خروج/تعویض حساب | پایلوت |

### 13.2 Unit Testهای اجباری

- Trim و Normalize نام کاربری ساده و رد UPN، DOMAIN\user، حساب Local و Control Character؛
- رد Username/Password null یا empty، طول بیش‌ازحد و تولید canonical Rate Key یکسان برای شکل‌های معادل؛
- اثبات اینکه Password حتی یک نویسه تغییر نمی‌کند؛
- تمام شاخه‌های Status Gate و مرز دقیق انقضا؛
- Password Never Expires، Password Expired، Must Change Password و مقادیر FILETIME ویژه بدون Overflow؛
- Upsert کاربر بر اساس AdObjectGuid، حفظ Identity پس از Rename و واگذاری SamAccountName قدیمی به objectGUID جدید؛
- تولید JWT، Claims اجباری، الگوریتم، type، issuer، audience و ClockSkew؛
- رد token بدون exp، sid، jti یا sub؛
- Idle و Absolute Expiry؛
- Refresh rotation، generation، grace و reuse خارج از grace؛
- Logout و Revocation reason؛
- Rate Limit بر اساس HMAC و IP؛
- Atomicity چند شکست هم‌زمان، Reset پس از موفقیت، پایان Cooldown، Recycle و Rotation کلید HMAC؛
- Generic error mapping و جلوگیری از Account Enumeration؛
- Local URL validation و جلوگیری از Open Redirect؛
- Options validation و Fail Fast؛
- Audit redaction و نبود Password، Token و Cookie.

### 13.3 Integration Testهای SQL و Application

- ساخت دیتابیس خالی و اجرای Migration Bundle؛
- اعمال Constraint و Unique Indexها؛
- هم‌زمانی دو Upsert برای یک AdObjectGuid؛
- هم‌زمانی چند Refresh با RowVersion و Transaction؛
- قرارداد 409/Retry-After داخل Grace و عدم Set-Cookie در درخواست بازنده؛
- Logout و استفاده مجدد از JWT کپی‌شده؛
- Logout با Access منقضی و Refresh معتبر؛
- Refresh با Access منقضی، Antiforgery قدیمی و Refresh معتبر از طریق Refresh scheme؛
- نشست Revoked، Idle Expired، Absolute Expired و User mismatch؛
- قطعی و Timeout SQL با نتیجه Fail Closed؛
- عدم صدور Cookie اگر Transaction نشست شکست بخورد؛
- Cleanup نشست، Refresh Token و Audit منقضی؛
- پایداری Data Protection بعد از Recycle؛
- state/nonce منقضی، callback مستقیم، replay و مصرف هم‌زمان؛
- سازگاری Release N و N−1 برای Schema، Cookie، key IDs و Session؛
- Security Headerها و Cache-Control روی صفحات Auth؛
- عدم پذیرش Authorization Header در browser JWT scheme.

### 13.4 آزمون واقعی Active Directory

برای هر حساب تست، Login دستی و SSO در صورت کاربرد اجرا می‌شود:

- Valid active account؛
- Wrong password؛
- User not found؛
- Disabled؛
- Locked؛
- Account expired؛
- Password expired؛
- Must change password؛
- Password never expires؛
- Fine-Grained Password Policy؛
- Rename شده با همان objectGUID.

شواهد لازم:

- هر Submit ناموفق حداکثر یک Bad Password Attempt در AD ایجاد کند؛
- شمارش Attempt از Security Event همان Writable DC متصل انجام شود، نه از badPwdCount تجمیعی نامطمئن؛
- Credential validation از Negotiate + Signing + Sealing استفاده کند؛
- LDAP Simple Bind بدون TLS مشاهده نشود؛
- gMSA فقط Read لازم را داشته باشد؛
- قطع DNS/DC هیچ Offline Login ایجاد نکند؛
- Credential validation هیچ Retry خودکار نداشته باشد و Snapshot همان DC را استفاده کند؛
- Blackhole DC، Bulkhead/Circuit Breaker را بدون Thread growth نامحدود فعال کند؛
- بازگشت DC بدون Restart برنامه قابل بازیابی باشد؛
- اختلاف ساعت کنترل‌شده Kerberos را Fail کند و علت زیرساختی مشخص باشد.

### 13.5 آزمون IIS، Kerberos و Chrome

1. <code>Test-ADServiceAccount</code> موفق باشد.
2. SPN دقیقاً یک مالک داشته و Duplicate نداشته باشد.
3. GPO در <code>chrome://policy</code> با Source سازمانی و Status=OK دیده شود.
4. درخواست مسیر عادی بدون JWT هیچ <code>WWW-Authenticate: Negotiate</code> ندهد.
5. فقط endpoint SSO Challenge ویندوز ایجاد کند.
6. کاربر پایلوت بدون Credential Prompt وارد شود.
7. <code>klist</code> Ticket برابر <code>HTTP/&lt;portal-fqdn&gt;</code> نشان دهد.
8. Windows Security Log یا Capture، Kerberos و عدم استفاده NTLM را ثابت کند.
9. تلاش اجباری NTLM در مسیر SSO رد شود و هیچ نشست/JWT دریافت نکند.
10. دسترسی با IP یا alias تأییدنشده موفق تلقی نشود.
11. Logout ویندوز را تغییر ندهد و ورود دستی با حساب دوم ممکن باشد.
12. دکمه Windows دوباره همان حساب جاری Windows را وارد کند.
13. Recycle App Pool تنظیمات و کلیدها را از بین نبرد.

### 13.6 آزمون امنیت

- Login CSRF، Logout CSRF، Refresh CSRF و Windows-login CSRF؛
- SameSite، Secure، HttpOnly، Host prefix و نبود Domain attribute؛
- Token tampering، الگوریتم none، algorithm confusion، kid نامعتبر و گواهی قدیمی؛
- issuer، audience، type و lifetime اشتباه؛
- Replay Access Token پس از Logout؛
- Refresh reuse داخل و خارج Grace؛
- Session fixation و استفاده مجدد sid؛
- Username Enumeration از طریق متن، status، redirect و timing؛
- Brute Force، Password Spraying و حمله توزیع‌شده محدود؛
- Rate Limit با درخواست‌های هم‌زمان، Client IP مستقیم، Reset پس از Success و مقادیر واقعی Lockout Policy؛
- ورودی بیش‌ازحد بزرگ، Unicode control، LDAP/filter injection و header injection؛
- Open Redirect و Host Header manipulation؛
- XSS روی DisplayName و Username؛
- Clickjacking و CSP با <code>frame-ancestors 'self'</code> یا <code>'none'</code>؛
- HSTS، X-Content-Type-Options، Referrer-Policy و محدودکردن cache؛
- بررسی Log، SQL، HTML و Browser Storage برای نبود Credential/Token؛ Crash dump خودکار در Production غیرفعال است و هر Dump مجاز بالقوه حاوی Password حافظه Managed تلقی، محدود، محافظت و سریع حذف می‌شود؛
- قطع SQL و AD در مراحل مختلف Login/Refresh؛
- Security Review مستقل پیش از Production به‌سبب Token Service سفارشی.

### 13.7 E2E تجربه کاربر

- اولین مراجعه و SSO خودکار؛
- Login دستی پس از شکست کنترل‌شده SSO؛
- Logout، نمایش فرم و ورود با حساب دیگر؛
- فعال‌کردن مجدد Windows SSO؛
- بستن کامل Chrome و رفتار NoSso؛
- چند Tab و Refresh هم‌زمان؛
- فرم بازمانده بیش از ۹ دقیقه و ۳۰ ثانیه و بیش از ۳۰ دقیقه؛
- پیام‌های فارسی، Keyboard navigation و Focus؛
- 401، 429، 503 و خطای عمومی با Correlation ID؛
- نبود Token در Local Storage، Session Storage و JavaScript.

### 13.8 پوشش و شواهد

- پوشش خط برای Domain/Application به‌تنهایی معیار کافی نیست؛ تمام Branchهای امنیتی باید تست نام‌دار داشته باشند.
- Build در صورت شکست هر تست Unit/Integration متوقف می‌شود.
- E2E و AD tests با Tag جدا و فقط در محیط مجاز اجرا می‌شوند.
- خروجی تست، نسخه Artifact، Hash فایل پارامتر و زمان اجرا در Evidence Package ثبت می‌شود.
- هیچ Evidence Package نباید Password یا Token خام داشته باشد.

---

## 14. ظرفیت و کارایی

### 14.1 پروفایل آزمون

| سناریو | هدف اولیه |
|---|---|
| بار عادی | ۱٬۰۰۰ Virtual User فعال، نه صرفاً ۱٬۰۰۰ رکورد Session |
| Stress | ۱٬۵۰۰ Virtual User فعال |
| Ramp | افزایش تدریجی در ۱۵ دقیقه |
| Soak | حداقل ۳۰ دقیقه در بار هدف |
| Login storm شبیه‌سازی‌شده | ۱۰ Login در ثانیه برای ۵ دقیقه با Fake AD |
| Real AD login test | محدود، با تأیید AD Admin و حساب‌های تست |
| Refresh storm | Refreshهای دارای jitter و چند Tab |
| Protected SSR traffic | ترکیب صفحه اصلی، me، Refresh و Logout |

تست ۱٬۵۰۰ Password واقعی روی AD بدون هماهنگی مدیر دامین ممنوع است؛ می‌تواند Lockout و فشار غیرمجاز ایجاد کند. مسیر Application با Fake AD در ظرفیت کامل و مسیر واقعی AD با سقف کنترل‌شده آزمایش می‌شود.

هر Virtual User نشست واقعی نگه می‌دارد و با Think Time تصادفی ۱۰ تا ۳۰ ثانیه، ترکیب ۷۰٪ صفحه محافظت‌شده، ۱۵٪ me، ۱۰٪ Refresh و ۵٪ Logout/Login را اجرا می‌کند. هدف اولیه حدود ۷۰ درخواست در ثانیه پایدار و Burst برابر ۱۵۰ درخواست در ثانیه است؛ نرخ نهایی پس از مشاهده الگوی واقعی سازمان بازتنظیم می‌شود. ایجاد Session بدون ترافیک فعال، معیار قبولی نیست.

### 14.2 معیار پذیرش اولیه

این مقادیر پیش از آزمون با سخت‌افزار واقعی بازبینی می‌شوند:

- p95 پاسخ صفحه محافظت‌شده کمتر از ۲ ثانیه؛
- p99 کمتر از ۴ ثانیه؛
- نرخ خطای غیرمنتظره کمتر از ۰٫۵٪؛
- عدم Deadlock SQL، Thread Pool starvation و Connection Pool exhaustion؛
- عدم ابطال اشتباه نشست در Refresh هم‌زمان؛
- عدم افزایش خطی غیرقابل کنترل latency ناشی از lookup sid؛
- CPU پایدار زیر ۸۰٪ و نبود Memory Paging مداوم؛
- Disk latency و SQL wait قابل قبول در Soak؛
- Audit write نباید گلوگاه کارایی شود؛ ثبت Audit موفقیت با تغییر Session اتمیک است و شکست Transaction مانع صدور نشست می‌شود؛
- AD Bulkhead در اشباع 429/503 کنترل‌شده بدهد و Queue حاوی Password نسازد.

### 14.3 تصمیم ظرفیت

قبولی Load Test به معنی High Availability نیست. اگر یک سرور نتواند معیارها را پاس کند، پیش از Go-Live یکی از این اقدامات لازم است: افزایش منابع، جداسازی SQL یا بازکردن تغییر محدوده برای Scale-out. کاهش معیار ۱٬۵۰۰ بدون تأیید مالک محصول مجاز نیست.

---

## 15. نقشه راه اجرایی

### 15.1 وابستگی فازها

    P0.1 تثبیت خط مبنا
          |
          v
    P0.2 Integration و Preflight
          |
          +----> P1 Foundation/Configuration
                        |
                        +----> P2 Data/Audit
                        |
                        +----> P3 Manual AD
                                   |
                      P2 + P3 ----> P4 JWT/Session/Refresh
                                      |
                      P0.2 + P4 ------> P5 Kerberos SSO/Logout
                                      |
                      P2..P5 -------> P6 UI/Errors
                                      |
                      P1..P6 -------> P7 Hardening/Tests
                                      |
                                      v
                               P8 Performance
                                      |
                                      v
                               P9 Pilot/Go-Live

### 15.2 WBS سطح اجرایی

| فاز | فعالیت‌های اصلی | خروجی و Gate خروج | نفرروز تقریبی همه نقش‌ها |
|---|---|---|---:|
| P0.1 — Baseline | تأیید Scope، ADRها، فرض‌ها و ریسک‌ها | همین سند امضاشده و Change Control | در انتظار Sign-off |
| P0.2 — Integration/Infra | مقادیر واقعی، Site/DB/FQDN/gMSA تست، DNS/TLS، SPN، IIS، Chrome GPO پایلوت، Preflight و Probe موقت | ورود Kerberos آزمایشی و محیط جدا | ۷–۱۱ |
| P1 — Solution Foundation | Solution، لایه‌ها، Config schema، Options validation، Fake AD، scripts پایه، Error/Correlation | Build تکرارپذیر و Fail-fast | ۴–۶ |
| P2 — Persistence/Audit | EF model، Users، Sessions، Refresh، LoginAttemptState، SsoChallenge، Audit، Index، expand-only migration، cleanup | Migration و concurrency tests پاس | ۵–۷ |
| P3 — Manual AD | Credential validator، Directory snapshot، Status Gate، rate limit، bulkhead، user sync | همه حساب‌های تست نتیجه صحیح | ۷–۱۰ |
| P4 — JWT/Session | Certificate/HMAC loader، JWT و Refresh schemes، result handler، sid lookup، rotation/replay، CSRF، Data Protection و N/N−1 compatibility | JWT/Refresh security suite پاس | ۱۰–۱۵ |
| P5 — SSO/Logout | IIS scheme isolation، state/nonce، identity mapping، session exchange، NoSso، Kerberos-only | SSO و account switch E2E پاس | ۶–۹ |
| P6 — Blazor UI | Login RTL، home، errors، accessibility پایه، refresh coordinator | UAT رابط و مسیرها پاس | ۴–۶ |
| P7 — Hardening/QA | Unit/Integration/E2E، security headers، enumeration، injection، failure tests، review مستقل | هیچ finding بحرانی/بالا باز نباشد | ۸–۱۲ |
| P8 — Capacity | load scripts، ۱٬۰۰۰/۱٬۵۰۰، login/refresh storm، tuning SQL/IIS | گزارش ظرفیت و Gate سخت‌افزار | ۴–۷ |
| P9 — Deploy/Handoff | Integration release، pilot، Production، smoke، rollback rehearsal، runbooks، evidence | Sign-off فنی و UAT | ۵–۸ |

### 15.3 برآورد کل

| نقش | تلاش تقریبی |
|---|---:|
| توسعه‌دهنده ارشد .NET | ۴۴–۶۱ نفرروز |
| مدیر AD/IIS/Windows/SQL | ۸–۱۳ نفرروز |
| QA، Security Review و UAT | ۸–۱۷ نفرروز |
| جمع پایه | ۶۰–۹۱ نفرروز |
| ذخیره ریسک برنامه‌ریزی | ۱۵٪ تا ۲۰٪ |
| بازه قابل برنامه‌ریزی | حدود ۶۹–۱۱۰ نفرروز |

برای یک توسعه‌دهنده ارشد با دسترسی موردی مدیر زیرساخت، زمان تقویمی محافظه‌کارانه ۱۳ تا ۱۸ هفته است. صدور مجوزها، گواهی، gMSA، GPO و پنجره تغییر می‌تواند تقویم را افزایش دهد. این تخمین تعهد قراردادی نیست و پس از P0.2 با داده واقعی بازبرآورد می‌شود.

### 15.4 فعالیت‌های هر فاز

#### P0.2 — Integration و زیرساخت

1. جایگزینی تمام Placeholderها؛
2. Preflight و گزارش Gap؛
3. ایجاد Site/App Pool/DB/FQDN آزمایشی؛
4. ایجاد و نصب gMSA؛
5. ثبت و کنترل SPN؛
6. TLS Binding و ACL کلید؛
7. Chrome GPO پایلوت؛
8. استقرار یک Probe حداقلی و موقت فقط در Site پایلوت برای Windows Identity؛
9. اثبات Kerberos و رد NTLM؛
10. حذف Probe تشخیصی یا محدودسازی کامل آن پیش از ادامه؛
11. Snapshot تنظیمات برای Rollback تغییر؛
12. تأیید Gate فاز.

#### P1 — Foundation

1. ایجاد Repository و Solution؛
2. تنظیم build deterministic و analyzers؛
3. پیاده‌سازی فایل پارامتر و schema؛
4. تعریف Options و ValidateOnStart؛
5. Fake AD فقط برای Development؛
6. middleware خطا، Correlation و Security headers؛
7. قراردادهای Application و test harness.

#### P2 — Data

1. Entity و mapping شامل LoginAttemptState و SsoChallenge؛
2. Index و concurrency؛
3. Migration Bundle؛
4. repositories/query services کمینه؛
5. Audit writer پاک‌سازی‌شده؛
6. cleanup job؛
7. integration tests.

#### P3 — Manual AD

1. Username validator؛
2. PrincipalContext factory؛
3. Credential validator؛
4. Directory snapshot و computed attributes؛
5. Status Gate؛
6. rate limit/HMAC/bulkhead؛
7. sync User؛
8. فرم Manual و error mapping؛
9. real AD tests.

#### P4 — JWT و نشست

1. Threat Model و ADR نهایی؛
2. X509 loader و rotation؛
3. JWT profile و validator؛
4. Cookie extraction، Refresh authentication scheme و منع Header؛
5. Session issuer؛
6. Refresh generation/HMAC/rotation؛
7. multi-tab grace و replay revoke؛
8. sid validation؛
9. Anti-forgery/Origin/redirect؛
10. HTML/API authorization result handler؛
11. Data Protection و HMAC key lifecycle؛
12. logout، cleanup و N/N−1 compatibility؛
13. security tests.

#### P5 — SSO

1. IIS Windows policy فقط برای callback؛
2. state/nonce و SSO start؛
3. Windows Identity mapping؛
4. Status Gate مشترک؛
5. Session issuer مشترک؛
6. NoSso و account switch؛
7. Chrome GPO pilot؛
8. Kerberos evidence.

#### P6 تا P9

- تکمیل UI فارسی و Errorها؛
- اجرای تست امنیتی و اصلاح findingها؛
- اجرای تست ظرفیت پیش از Go-Live؛
- استقرار مرحله‌ای Integration، Pilot و Production؛
- تحویل Source، Artifact، scripts، ADR، Runbook و Evidence.

---

## 16. استقرار و Rollback

### 16.1 Artifactهای Release

هر Release باید شامل موارد زیر باشد:

- Artifact منتشرشده Release و فایل Hash؛
- نسخه دقیق .NET Runtime/Hosting Bundle موردنیاز؛
- فایل پارامتر محیط بدون Secret؛
- JSON Schema تنظیمات؛
- Migration Bundle یا SQL script idempotent؛
- اسکریپت Preflight و Drift Check؛
- اسکریپت Configure-IIS؛
- اسکریپت Validate-SPN/gMSA/Certificate/GPO؛
- Smoke Test؛
- Release Notes، ADRهای مرتبط و Known Issues؛
- نسخه قبلی قابل بازگشت؛
- Evidence template.

### 16.1.1 قرارداد سازگاری N و N−1

- Schema migration، config schema، Cookie names، JWT type/claims، signing validation keys، HMAC key IDs و Data Protection application name باید میان Release جدید و نسخه قبلی Rollback-compatible باشند.
- نسخه N−1 باید بتواند حداقل Token/Session ساخته‌شده توسط N را رد یا به‌صورت امن اعتبارسنجی کند و نباید Startup آن به‌دلیل KeyId جدید Fail شود.
- Private Key و HMAC validation keyهای قبلی تا پایان Change/Rollback window حذف نمی‌شوند.
- Rollback rehearsal شامل یک نشست ساخته‌شده توسط نسخه N و اجرای نسخه N−1 است.
- اگر Compatibility عمداً شکسته شود، Release Notes باید Rollback را همراه با ابطال تمام نشست‌ها و Re-login اجباری تعریف کند؛ Rollback خاموش با نشست‌های قدیمی مجاز نیست.

### 16.2 ترتیب استقرار Integration

1. اجرای Preflight بدون تغییر و رفع تمام خطاها؛
2. اجرای Unit و Integration tests؛
3. ساخت Artifact یک‌بار و استفاده از همان Artifact در همه محیط‌ها؛
4. اعمال Migration سازگار با نسخه قبلی؛
5. استقرار Package در Site آزمایشی؛
6. اعمال ACLهای gMSA، Certificate و Data Protection؛
7. اعمال IIS Desired State؛
8. اجرای Smoke Test دستی و خودکار؛
9. اجرای AD account matrix؛
10. اجرای Chrome/Kerberos E2E؛
11. اجرای Security suite؛
12. اجرای Load Test پیش از Production؛
13. تمرین Rollback؛
14. ثبت Evidence و تأیید Gate.

### 16.3 ترتیب Go-Live

1. Freeze Artifact، Hash و فایل پارامتر؛
2. تأیید Change Window و مسئولان حاضر؛
3. تأیید مجدد ریسک نبود Backup و Monitoring؛
4. اجرای Preflight و توقف در صورت هر خطای بحرانی؛
5. نگهداری نسخه قبلی Artifact و Snapshot تنظیمات IIS/GPO/SPN؛
6. اعمال Migration expand-only؛
7. استقرار برنامه و Recycle کنترل‌شده App Pool؛
8. Smoke Test ورود دستی؛
9. Smoke Test SSO با OU پایلوت؛
10. اثبات Kerberos و بررسی نبود NTLM؛
11. Smoke Test Logout، تعویض حساب، Refresh و استفاده از Token باطل؛
12. گسترش مرحله‌ای GPO از پایلوت به کاربران؛
13. UAT محدود؛
14. ثبت نتیجه و Sign-off.

اعمال GPO عمومی پیش از موفقیت کامل ورود دستی و Rollback آزمایشی ممنوع است.

### 16.4 محرک‌های Rollback

- افزایش گسترده 401/403 یا Credential Prompt؛
- مشاهده NTLM؛
- SPN duplicate یا <code>KRB_AP_ERR_MODIFIED</code>؛
- توقف App Pool یا خطای Startup configuration؛
- شکست ورود دستی و SSO هم‌زمان؛
- خطای Migration یا عدم سازگاری Schema؛
- خطای Refresh گسترده یا ابطال اشتباه نشست؛
- یافته امنیتی Critical/High؛
- عدم عبور Smoke Test در زمان مقرر Change Window.

### 16.5 ترتیب Rollback

1. Feature Flag مربوط به Auto SSO غیرفعال و ورود دستی حفظ شود.
2. گسترش GPO متوقف و Link پایلوت در صورت نیاز غیرفعال شود.
3. Artifact قبلی بازگردانده شود.
4. Snapshot تنظیمات IIS بازگردانده و App Pool Recycle شود.
5. Binding یا Certificate فقط در صورت ارتباط مستقیم با خطا بازگردانده شود.
6. SPN جدید فقط با اثبات مالکیت قبلی و Change Record حذف یا بازگردانده شود.
7. gMSA و KDS Root Key در Rollback فوری حذف نشوند.
8. Schema به‌دلیل Expand-only بودن Rollback نمی‌شود؛ نسخه قبلی باید آن را تحمل کند.
9. Smoke Test ورود دستی و نبود Windows Challenge در مسیرهای عادی اجرا شود.
10. Incident و Evidence ثبت و Go-Live متوقف شود.

Snapshot تنظیمات و نگهداری Artifact قبلی بخشی از ایمنی Change است و جایگزین Backup داده، که خارج از محدوده اعلام شده، نیست.

---

## 17. مسئولیت‌ها

| نقش | مسئولیت |
|---|---|
| مالک محصول | تأیید Scope، پذیرش ریسک‌ها، UAT و Changeها |
| توسعه‌دهنده ارشد .NET | طراحی، کدنویسی، تست، Migration، scripts و مستندات |
| مدیر AD/GPO | مقادیر دامین، حساب‌های تست، gMSA، SPN، Lockout Policy و Chrome GPO |
| مدیر Windows/IIS | IIS Roles، Hosting Bundle، Site، App Pool، TLS و ACL |
| مدیر SQL | Database، Login gMSA، منابع، Migration Window و بررسی Queryها |
| بازبین امنیت | Threat Model، JWT-in-Cookie، CSRF، Crypto و یافته‌های Security |
| نماینده کاربران | آزمون SSO، Logout، تعویض حساب و پیام‌های UI |

با فرض یک توسعه‌دهنده، مدیران AD/IIS/SQL باید در P0.2، P3، P5، P8 و P9 در زمان از پیش رزروشده در دسترس باشند. نبود دسترسی زیرساختی، Blocker تقویمی است.

---

## 18. ماتریس ریسک

| ریسک | احتمال | اثر | کنترل | ریسک باقیمانده |
|---|---|---|---|---|
| JWT-in-Cookie سفارشی برخلاف الگوی ساده Cookie/BFF | متوسط | زیاد | ADR، جداسازی Issuer، CSRF، review و تست گسترده | متوسط |
| وابستگی هر درخواست به SQL برای sid | متوسط | زیاد | PK/Index، NoTracking، Load Test و Fail Closed | متوسط |
| Single Point of Failure سرور | متوسط | زیاد | Runbook و استقرار کنترل‌شده؛ HA خارج Scope | زیاد و پذیرفته‌شده |
| نبود Backup | متوسط | بسیار زیاد | Migration فقط expand-only؛ پذیرش صریح | بسیار زیاد و پذیرفته‌شده |
| نبود Monitoring | زیاد | زیاد | Audit و گزارش کاربر؛ ابزار پایش خارج Scope | زیاد و پذیرفته‌شده |
| رقابت IIS و SQL روی یک سرور | زیاد | زیاد | Load Test، sizing و tuning | متوسط تا زیاد |
| قفل حساب AD از طریق ورود دستی | متوسط | زیاد | Rate Limit قبل AD، یک attempt، هماهنگی Lockout Policy | متوسط |
| بازگشت Negotiate به NTLM | متوسط | زیاد | Kerberos evidence، NTLM block پس از impact assessment | کم تا متوسط |
| Credential Prompt بومی Chrome هنگام شکست Kerberos | متوسط | متوسط | SPN/GPO/زمان صحیح، پایلوت و مسیر دستی پس از لغو Prompt | متوسط |
| SPN اشتباه یا تکراری | متوسط | زیاد | setspn query، Preflight و gMSA ownership | کم |
| Config drift میان فایل، IIS، AD و GPO | متوسط | زیاد | Drift Check و Evidence | متوسط |
| قطعی AD | متوسط | متوسط | Fail Closed؛ بدون Offline credential | متوسط و پذیرفته‌شده |
| قطعی SQL | متوسط | زیاد | Fail Closed؛ single server risk | زیاد و پذیرفته‌شده |
| Race چند Tab در Refresh | متوسط | زیاد | Transaction، rowversion، grace و E2E | کم تا متوسط |
| Replay Token سرقت‌شده | کم | زیاد | HttpOnly، TLS، سقف ۱۰ دقیقه با ClockSkew، sid revocation | کم تا متوسط |
| CSRF | متوسط | زیاد | Anti-forgery، Strict، Origin و POST-only | کم |
| Username Enumeration | متوسط | متوسط | پیام/status/timing عمومی و Audit داخلی | کم |
| Session Cookie بازیابی‌شده Chrome | کم تا متوسط | متوسط | E2E و دکمه صریح Windows SSO | متوسط |
| محیط Integration روی همان سرور | متوسط | متوسط | Site/DB/FQDN جدا و تست پیش از Go-Live | متوسط و پذیرفته‌شده |
| APIهای همگام DirectoryServices زیر بار | متوسط | زیاد | Bulkhead، Queue کوتاه و Load Test | متوسط |
| انقضای گواهی بدون Alert | متوسط | زیاد | Runbook دستی؛ Monitoring خارج Scope | زیاد و پذیرفته‌شده |
| نسخه Patch نشده .NET/Windows | متوسط | زیاد | Preflight و الزام نسخه پشتیبانی‌شده | متوسط |

---

## 19. کنترل تغییر محدوده

موارد زیر نیازمند Change Request، تحلیل امنیت، بازبرآورد و ADR جدید هستند:

- افزودن مرورگر، موبایل، اینترنت یا دستگاه خارج از دامین؛
- پذیرفتن NTLM؛
- افزودن Role/Group/OU؛
- MFA یا Identity Provider؛
- API مستقل یا CORS؛
- چند IIS یا Load Balancer؛
- انتقال SQL به سرور دیگر؛
- تغییر JWT-in-Cookie به Cookie/BFF یا برعکس؛
- تغییر SameSite از Strict؛
- ذخیره Token در JavaScript Storage؛ این مورد به‌صورت پیش‌فرض رد می‌شود؛
- تغییر به Migration مخرب؛
- اضافه‌کردن Backup یا Monitoring؛
- تغییر سیاست Session، Lockout یا ظرفیت؛
- فعال‌کردن Interactive Server به‌صورت گسترده.

هر Change Request باید اثر بر Scope، Threat Model، Test Matrix، IIS/AD/GPO و زمان‌بندی را مشخص کند.

---

## 20. Definition of Ready برای شروع پیاده‌سازی

کدنویسی Production-grade زمانی شروع می‌شود که موارد زیر تأیید شوند:

- [ ] این سند به‌عنوان Scope Baseline پذیرفته شده باشد.
- [ ] نام واقعی AD، NetBIOS، FQDN، SQL و gMSA مشخص باشد.
- [ ] Windows Server و SQL Server version/edition پشتیبانی‌شده تأیید شوند.
- [ ] سخت‌افزار سرور ثبت شود.
- [ ] Lockout Policy واقعی AD دریافت شود.
- [ ] ساخت محیط Integration، حساب‌های تست و OU پایلوت مجاز باشد.
- [ ] دسترسی مدیران AD/IIS/SQL در فازهای لازم رزرو شود.
- [ ] گواهی TLS و امکان ساخت گواهی JWT وجود داشته باشد.
- [ ] امکان ساخت gMSA، SPN و Chrome GPO تأیید شود.
- [ ] ریسک نبود Backup، Monitoring و HA رسماً پذیرفته شود.
- [ ] Repository و روش نگهداری Source تعیین شود.
- [ ] معیارهای کارایی و UAT مالک مشخص داشته باشند.

Placeholderها مانع Scaffold و Fake-AD development نیستند، اما P0.2 واقعی و Go-Live بدون تکمیل این فهرست مجاز نیست.

---

## 21. Definition of Done

- [ ] کد Build و تمام Unit/Integration tests سبز است.
- [ ] هیچ Secret یا Credential در Repository، Artifact، Log و SQL نیست.
- [ ] Migration از دیتابیس خالی و موجود موفق و expand-only است.
- [ ] Manual AD matrix کامل پاس شده است.
- [ ] Kerberos با شواهد واقعی و بدون NTLM تأیید شده است.
- [ ] SSO، Logout، NoSso، تعویض حساب و Windows re-entry در Chrome پاس شده‌اند.
- [ ] JWT tampering، replay، logout revocation، refresh reuse و key rotation پاس شده‌اند.
- [ ] CSRF، Open Redirect، Enumeration، Rate Limit و Failure tests پاس شده‌اند.
- [ ] تست ۱٬۰۰۰ و Stress ۱٬۵۰۰ معیارها را پاس کرده‌اند.
- [ ] Rollback rehearsal موفق است.
- [ ] Production parameter file و Drift Check بدون خطاست.
- [ ] Security finding با شدت Critical یا High باز نیست.
- [ ] Runbook و Evidence Package تحویل شده‌اند.
- [ ] UAT و Sign-off مالک محصول ثبت شده‌اند.

---

## 22. تصمیم‌های معماری ثبت‌شده

| ADR | تصمیم |
|---|---|
| ADR-001 | Windows Authentication فقط برای endpoint SSO؛ JWT scheme پیش‌فرض |
| ADR-002 | Kerberos الزامی و NTLM غیرقابل قبول در Production |
| ADR-003 | ورود دستی با sAMAccountName و PrincipalContext دارای Signing/Sealing |
| ADR-004 | JWT کوتاه‌عمر داخل Secure HttpOnly Cookie؛ تصمیم سفارشی و آگاهانه |
| ADR-005 | Session state و sid lookup در SQL برای Logout فوری |
| ADR-006 | Refresh Token opaque، HMAC شده، rotating و reuse-aware |
| ADR-007 | Static SSR برای Authentication؛ Interactive Server فقط آمادگی آینده |
| ADR-008 | gMSA و Integrated Security برای SQL/AD/Certificate access |
| ADR-009 | فایل Desired State واحد، با Configuration Planeهای جدا |
| ADR-010 | Single Server و نبود HA در فاز اول |
| ADR-011 | Backup و Monitoring خارج از محدوده و ریسک پذیرفته‌شده |
| ADR-012 | Migration فقط Expand-only تا زمانی که Backup خارج Scope است |
| ADR-013 | Fail Closed در قطعی AD برای Login/Refresh و قطعی SQL برای Authorization |

---

## 23. منابع رسمی مرجع

- [.NET support policy و وضعیت .NET 10 LTS](https://dotnet.microsoft.com/en-us/platform/support/policy/dotnet-core)
- [میزبانی ASP.NET Core روی IIS](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/?view=aspnetcore-10.0)
- [Windows Authentication در ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/windowsauth?view=aspnetcore-10.0)
- [IISServerOptions و AutomaticAuthentication](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/in-process-hosting?view=aspnetcore-10.0)
- [IIS Application Initialization](https://learn.microsoft.com/en-us/iis/configuration/system.webserver/applicationinitialization/)
- [مدیریت gMSA در Windows Server](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)
- [فرمان setspn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/setspn)
- [عیب‌یابی Kerberos در IIS](https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/iis/www-authentication-authorization/troubleshoot-kerberos-failures-ie)
- [Chrome AuthServerAllowlist](https://chromeenterprise.google/policies/#AuthServerAllowlist)
- [Chrome policy templates](https://support.google.com/chrome/a/answer/7649838)
- [PrincipalContext.ValidateCredentials](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.principalcontext.validatecredentials)
- [ContextOptions](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.contextoptions)
- [UserPrincipal و وضعیت حساب](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.userprincipal)
- [IsAccountLockedOut](https://learn.microsoft.com/en-us/dotnet/api/system.directoryservices.accountmanagement.authenticableprincipal.isaccountlockedout)
- [msDS-User-Account-Control-Computed](https://learn.microsoft.com/en-us/windows/win32/adschema/a-msds-user-account-control-computed)
- [Rate Limiting در ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit?view=aspnetcore-10.0)
- [رمزنگاری و اعتبارسنجی گواهی SqlClient](https://learn.microsoft.com/en-us/sql/connect/ado-net/encryption-and-certificate-validation?view=sql-server-ver17)
- [JWT Bearer در ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-jwt-bearer-authentication?view=aspnetcore-10.0)
- [Blazor Authentication و Anti-forgery](https://learn.microsoft.com/en-us/aspnet/core/blazor/security/?view=aspnetcore-10.0)
- [HttpContext در Blazor](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/httpcontext?view=aspnetcore-10.0)
- [SameSite Cookies در ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/security/samesite?view=aspnetcore-10.0)
- [Data Protection configuration](https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/configuration/overview?view=aspnetcore-10.0)
- [RFC 7519 — JWT](https://datatracker.ietf.org/doc/html/rfc7519)
- [RFC 8725 — JWT Security Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725.html)
- [RFC 9068 — JWT Access Token Profile](https://www.rfc-editor.org/rfc/rfc9068.html)
- [RFC 9700 — OAuth Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html)

---

## 24. نقطه تصمیم بعدی

پس از تأیید این سند، اولین اقدام اجرایی «P0.2 — جایگزینی Placeholderها و ساخت محیط Integration/Preflight» است. توسعه Featureهای واقعی AD و JWT پیش از آماده‌شدن این Gate فقط با Fake AD و دیتابیس Development مجاز خواهد بود.
