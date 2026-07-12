# قرارداد Evidence

هر مدرک باید به Requirement یا Work Item مشخص متصل باشد و حداقل شامل موارد
زیر باشد:

- Evidence ID؛
- فرمان یا روش بررسی دقیق؛
- working directory و target environment؛
- نسخه ابزار یا runtime مرتبط؛
- زمان UTC؛
- commit یا وضعیت Git؛
- exit code؛
- نتیجه PASS، FAIL، BLOCKED یا NOT RUN؛
- خلاصه redacted؛
- بازبین.

مدرک پس از تغییر مرتبط کد یا تنظیمات باید دوباره تولید شود. فایل خام حساس
در Git ذخیره نمی‌شود؛ فقط SHA-256، محل کنترل‌شده، مالک و retention ثبت می‌شود.
