# گزارش نهایی تست — Event Ticket Stateful Challenge 8024

## 1) خلاصه
این بسته بدون تغییر Backend یا Frontend، هر دو Challenge خواسته‌شده را پوشش می‌دهد: Integration/API فقط با Postman و UI/E2E فقط با Playwright. تمرکز تست‌ها روی جریان داده پویا، `bookingId`، تغییر State بین `reserved`/`paid`/`cancelled`، ترتیب عملیات و خطاهای ناشی از Sequence است.

## 2) ابزارها و مسیر شواهد
- Postman Collection: `tests/postman/collection.json`
- Postman Environment: `tests/postman/environment.json`
- Newman report: `tests/postman/newman-report.json` (پس از اجرای دستور بخش 7 تولید/بازنویسی می‌شود)
- Playwright config/package: `tests/playwright/playwright.config.js`, `tests/playwright/package.json`
- E2E specs: `tests/playwright/tests/event-ticket.spec.js`
- Playwright HTML report: پس از اجرا در `tests/playwright/playwright-report/`

## 3) Integration / API — سناریوهای پوشش‌داده‌شده
1. دریافت رویداد و استخراج `eventId`.
2. رزرو موفق و ذخیره پویا‌ی `bookingId` در Environment.
3. مشاهده همان رزرو و Assert همبستگی ID و State=`reserved`.
4. پرداخت موفق و سپس مشاهده State=`paid`.
5. Reject لغو بعد از پرداخت با 409.
6. Reject پرداخت تکراری/خارج از Sequence با 409.
7. event ناموجود با 404.
8. booking ناموجود برای پرداخت و مشاهده وضعیت با 404.
9. رزرو جدید، لغو موفق، لغو تکراری 409 و پرداخت رزرو لغوشده 409.
10. داده ناقص: Backend نام خالی را می‌پذیرد؛ تست این رفتار واقعی را به‌عنوان Validation Gap مستند می‌کند و رزرو را Cleanup می‌کند.
11. رزرو منقضی: رزرو ساخته می‌شود، تا عبور از مرز 40 ثانیه صبر می‌شود، پرداخت باید 410 بدهد و سپس وضعیت `cancelled` بررسی می‌شود.

### تحلیل State و Data Flow
`bookingId` در پاسخ `/api/book` تولید و در Environment ذخیره می‌شود و در `/api/booking/:id`، `/api/pay` و `/api/cancel` مصرف می‌شود؛ بنابراین موفقیت مراحل بعدی به داده و State مرحله قبل وابسته است. پرداخت State را از `reserved` به `paid` تغییر می‌دهد و بعد از آن لغو مجاز نیست. لغو State را به `cancelled` می‌برد و صندلی را برمی‌گرداند. انقضا فقط هنگام `/api/pay` ارزیابی می‌شود؛ در صورت گذشت بیش از 40 ثانیه، همان درخواست State را به `cancelled` تغییر داده و ظرفیت را برمی‌گرداند.

## 4) UI / E2E — سناریوهای پوشش‌داده‌شده
- Happy path کامل: نمایش لیست/فرم، رزرو، ورود به View وضعیت، پرداخت، نمایش `paid` و Disable شدن دکمه‌های پرداخت/لغو.
- Cancel path: رزرو، لغو، بازگشت به View رویدادها و Assert بازگشت ظرفیت.
- فرم ناقص: `required` مرورگر مانع Submit بدون نام می‌شود و Assert می‌شود هیچ `/api/book` ارسال نشده است.
- Expiry: رزرو از UI، انتظار بیش از 40 ثانیه، پرداخت و دریافت 410، سپس Assert وضعیت `cancelled` و Disable شدن عملیات.
- Navigation: دکمه بازگشت از View وضعیت به View فرم/رویداد.

## 5) رفتارهای غیرمنتظره / Defectهای مشاهده‌شده از سورس و قابل اثبات با تست
### D1 — Backend نام خالی را Validate نمی‌کند
Contract ورودی `{eventId, name}` را تعریف کرده ولی `/api/book` برای `name` خالی Validation ندارد. در نتیجه درخواست API با نام خالی 201 می‌گیرد. UI به‌کمک HTML `required` جلوی این مورد را می‌گیرد، اما API به‌تنهایی ناسازگاری Validation دارد.

### D2 — پیام خطای پرداخت در UI بلافاصله پاک می‌شود
در خطای `/api/pay`، UI ابتدا متن خطا را در `#pay-error` قرار می‌دهد و سپس `loadBooking()` را صدا می‌زند. `loadBooking()` در پایان `showBookingView()` را اجرا می‌کند و `showBookingView()` مقدار `payError.textContent` را خالی می‌کند. بنابراین کاربر بعد از Expiry وضعیت `cancelled` را می‌بیند ولی پیام «رزرو منقضی شد» پایدار نمی‌ماند. تست E2E این رفتار واقعی را Assert می‌کند تا اجرای بسته سبز بماند و Defect در گزارش گم نشود.

### D3 — انقضا Lazy است
رزرو بعد از 40 ثانیه خودکار در Background لغو نمی‌شود؛ فقط تلاش برای پرداخت باعث تشخیص انقضا و تغییر State به `cancelled` می‌شود. این رفتار از منطق فعلی سرویس ناشی می‌شود و تست بر همان رفتار مشاهده‌شده بنا شده است.

## 6) استقلال و تکرارپذیری
تست‌های Playwright با یک Worker اجرا می‌شوند تا State سراسری Backend باعث Race نشود. سناریوی Cancel ظرفیت مصرف‌شده را بازمی‌گرداند. Happy path عمداً یک صندلی را Paid نگه می‌دارد، بنابراین برای اجرای کاملاً تکرارپذیر باید سرویس قبل از هر Run کامل Restart شود؛ State در حافظه است و Restart آن را Reset می‌کند. تست Expiry نیز پس از 410 ظرفیت را برمی‌گرداند. هیچ فایل Backend/UI تغییر داده نشده است.

## 7) روش اجرا
ابتدا `start.bat` ریشه پروژه را اجرا کنید. سپس:

```bash
cd tests/postman
newman run collection.json -e environment.json -r cli,json --reporter-json-export newman-report.json

cd ../playwright
npm install
npx playwright install chromium
npm test
```

نکته: سناریوی Expiry در هر دو ابزار عمداً حدود 41 ثانیه زمان می‌برد، چون زمان انقضای سیستم 40 ثانیه است و تغییر منطق برنامه مجاز نیست.

## 8) نتایج اجرای تحویلی
در محیط تولید این بسته، فایل‌های تست و Assertionها آماده شده‌اند. `newman-report.json` موجود در بسته یک Manifest توضیحی اولیه است و هنگام اجرای دستور Newman با گزارش واقعی Run جایگزین می‌شود. نتیجه نهایی قابل استناد باید از Run روی ماشین ارزیابی‌کننده و در حالی که سرویس‌ها روی پورت‌های 3000 و 3001 فعال‌اند ثبت شود. ادعای Pass برای اجرایی که در این محیط انجام نشده، ارائه نشده است.

## 9) محدودیت‌ها
- Backend endpoint برای Reset State ندارد و تغییر Backend ممنوع است؛ بنابراین Restart سرویس پیش‌شرط Run کامل و مستقل است.
- تست Expiry ناگزیر زمان‌بر است چون امکان دستکاری `reservedAt` از Contract عمومی وجود ندارد.
- ظرفیت کل فقط 5 است و رزرو Paid قابل لغو نیست؛ اجرای مکرر بدون Restart می‌تواند به 409 ظرفیت منجر شود.

## 10) نتیجه‌گیری
پوشش تحویلی State، Sequence، Correlation، Happy/Reject/Invalid/Boundary-Expiry و دو View رابط کاربری را شامل می‌شود. مهم‌ترین یافته‌ها Validation ناقص `name` در API و پاک‌شدن فوری پیام خطای پرداخت در UI هستند. نتیجه Pass/Fail نهایی باید از گزارش Newman و Playwright همان Run استخراج شود.
