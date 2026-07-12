# EPortal

پرتال داخلی سازمان با احراز هویت Microsoft Active Directory، ورود خودکار
Kerberos SSO، ورود دستی دامین و نشست مبتنی بر JWT.

## وضعیت فعلی

- مرحله پروژه: آماده‌سازی Integration و Preflight
- فاز فعال: P0.2 — Draft و در انتظار تکمیل ورودی‌ها
- کد محصول: هنوز ایجاد نشده است
- تغییر زیرساخت: هنوز انجام نشده است

Scope Baseline نسخه ۱ با مرجع <code>APP-P0.1-001</code> تأیید شده است.
شروع‌نشدن کدنویسی در این وضعیت عمدی است؛ بسته P0.2 باید ابتدا تکمیل و
تصویب شود و هر تغییر واقعی زیرساخت نیز مجوز مستقل همان عملیات را نیاز دارد.

## اسناد اصلی

- [نقشه راه و معماری](docs/architecture/roadmap.md)
- [فراداده خط مبنای تأییدشده](docs/architecture/baseline-metadata.md)
- [راهنمای اجرای مرحله‌ای با عامل هوش مصنوعی](docs/governance/ai-implementation-playbook-fa.md)
- [وضعیت جاری پیاده‌سازی](docs/implementation/status.md)
- [بسته فاز P0.1](docs/phases/P0.1-baseline.md)
- [پیش‌نویس بسته فاز P0.2](docs/phases/P0.2-integration-preflight.md)
- [قرارداد عامل‌های پروژه](AGENTS.md)

## قاعده اجرا

در هر زمان فقط یک فاز فعال است. عبور به فاز بعد، اجرای تغییرات زیرساختی،
commit، push یا ایجاد Pull Request به تأیید صریح مالک پروژه نیاز دارد.
