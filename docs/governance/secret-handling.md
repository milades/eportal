# مدیریت Secret و شواهد حساس

## موارد ممنوع در Git، Prompt، Log و Evidence

- Password یا Credential دامین؛
- GitHub PAT یا هر access token؛
- JWT، Refresh Token، Cookie یا Anti-forgery value؛
- PFX، Private Key، HMAC key یا Data Protection key؛
- Connection String دارای Password؛
- Memory dump، database dump یا داده واقعی کاربران؛
- Windows Security Log یا packet capture خام؛
- فایل تنظیمات واقعی محیط وقتی حاوی topology محدود یا secret reference حساس است.

## روش مجاز

- در اسناد از Placeholder و شناسه Secret استفاده شود.
- Secret واقعی فقط از محل حفاظت‌شده محیط مقصد تزریق شود.
- متادیتای گواهی فقط پس از حذف اطلاعات غیرضروری ثبت شود؛ Private Key هرگز.
- Evidence داخل Git یک خلاصه redacted است.
- فایل خام لازم برای بررسی امنیتی، در storage کنترل‌شده نگهداری و فقط
  SHA-256، محل کنترل‌شده، مالک و retention آن در manifest ثبت می‌شود.
- پیش از commit و push باید diff و الگوهای Secret بررسی شوند.

## رخداد نشت

اگر Secret در خروجی یا Git مشاهده شد:

1. کار متوقف شود؛
2. مقدار در گزارش تکرار نشود؛
3. مالک Secret مطلع و Rotation انجام شود؛
4. اثر و محل انتشار بررسی شود؛
5. پاک‌سازی History فقط با طرح مصوب انجام شود؛
6. Incident و کنترل پیشگیرانه ثبت شود.
