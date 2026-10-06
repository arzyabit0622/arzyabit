# گزارش نهایی ارزیابی Performance / Scalability / Load / Stress سرویس Stateful Ticketing

## 1) هدف و دامنه تست
هدف این ارزیابی بررسی عملکرد، مقیاس‌پذیری و پایداری API رزرو بلیت Stateful روی `localhost:3000` با Apache JMeter است. مطابق Contract سرویس، سناریوی معتبر باید از توالی `register → tickets → reserve → pay/cancel` پیروی کند و خطاهای 4xx ناشی از ورودی یا State نامعتبر از خطاهای زیرساختی مانند 5xx، timeout و connection failure جدا تحلیل شوند.

در این گزارش، اعداد بخش نتایج مستقیماً از اسکرین‌شات‌های `Aggregate Report` و `Summary Report` اجرای واقعی JMeter استخراج شده‌اند. دو اجرای قابل استناد در اختیار است:

1. اجرای Load/Scalability برای Stageهای 10، 30، 50، 70 و 100 کاربر؛ مجموع کاربران این Stageها برابر 260 است.
2. اجرای Stress شامل 25 Thread و 8 Loop، یعنی 200 iteration و در اجرای مشاهده‌شده دقیقاً 1000 Sample.

> نکته مهم: مقادیر `Error %` در JMeter برای پاسخ‌های HTTP 4xx به‌صورت Failure ثبت می‌شوند، حتی وقتی همان 401/404 عمداً در Negative Test انتظار می‌رود. بنابراین Error% خام JMeter در این تست معادل «خرابی سیستم» نیست و باید بر اساس نوع پاسخ تفسیر شود.

---

## 2) فایل‌ها و روش اجرا
فایل JMeter:

`tests/jmeter/test-plan.jmx`

اجرای سرویس:

`start.bat`

اجرای پیشنهادی غیر GUI برای خروجی قابل استناد:

```bash
jmeter -n -t tests/jmeter/test-plan.jmx -l tests/jmeter/results.jtl -e -o tests/jmeter/report
```

به علت Stateful بودن سرویس، قبل از Run مستقل بهتر است Process مربوط به Node.js Restart شود تا وضعیت بلیت‌ها، کاربران و Reservationهای Run قبلی روی Run بعدی اثر نگذارد.

---

## 3) طراحی سناریوی Load / Scalability
پلن تست دارای پنج Stage با تعداد کاربران زیر است:

| Stage | Concurrent Users |
|---|---:|
| Load-10 | 10 |
| Load-30 | 30 |
| Load-50 | 50 |
| Load-70 | 70 |
| Load-100 | 100 |
| **Total threads across stages** | **260** |

سناریوی طراحی‌شده شامل ثبت‌نام، دریافت بلیت‌ها، رزرو و سپس Pay یا Cancel است. همچنین Negative Path برای user نامعتبر، ticket نامعتبر و reservation نامعتبر تعریف شده است.

در گزارش واقعی ارسال‌شده، `01 Register` و `02 List Tickets` هرکدام دقیقاً 260 Sample دارند که با مجموع Threadهای پنج Stage برابر است و نشان می‌دهد همه کاربران تا این دو مرحله اجرا شده‌اند.

---

## 4) نتایج واقعی Load / Scalability (10 تا 100 User)
### 4.1 خلاصه کل Run
بر اساس Summary/Aggregate Report اجرای 10 تا 100 User:

| Metric | مقدار واقعی |
|---|---:|
| Total Samples | **1300** |
| Average Response Time | **0 ms** (گرد شده) |
| Median | **1 ms** |
| P90 | **1 ms** |
| P95 | **1 ms** |
| P99 | **2 ms** |
| Minimum | **0 ms** |
| Maximum | **2 ms** |
| Std. Deviation | **0.53 ms** |
| Raw JMeter Error % | **60.00%** |
| Total Throughput | **8.9 req/sec** |
| Received | **2.63 KB/sec** |
| Sent | **1.74 KB/sec** |
| Average response size | **301.6 bytes** |

### 4.2 Endpointهای اصلی موفق

| Label | Samples | Avg | Median | P95 | P99 | Max | Error % | Throughput |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| 01 Register | **260** | **1 ms** | **1 ms** | **2 ms** | **2 ms** | **2 ms** | **0.00%** | **1.8/sec** |
| 02 List Tickets | **260** | **0 ms** | **0 ms** | **1 ms** | **1 ms** | **1 ms** | **0.00%** | **1.8/sec** |

نتیجه این بخش مثبت است: در 520 درخواست اصلی مشاهده‌شده برای Register و List Tickets هیچ خطایی ثبت نشده و Tail Latency نیز بسیار پایین بوده است.

### 4.3 Negative Pathهای مشاهده‌شده
در Run ارسال‌شده، مجموع Sampleهای 4xx برابر است با:

`1300 total - 520 successful Register/List = 780 expected 4xx samples`

یعنی **60% کل Sampleها** پاسخ 4xx بوده‌اند. این دقیقاً با `Error % = 60.00%` در Summary Report منطبق است.

نمونه Labelهای مشاهده‌شده در گزارش:

- `401`
- `user-401`
- `401-user`
- `ticket reserve-404`
- `Pay-404`
- `N1 Invalid tickets user`
- `N2 Invalid ticket reserve`
- `N3 Pay unknown reservation`

برای همه این Negative Requestها، JMeter مقدار Error را 100% نشان می‌دهد؛ اما چون هدف سناریو دریافت 401/404 بوده، این خطاها باید **Business/Contract Error مورد انتظار** تلقی شوند، نه Failure زیرساخت.

### 4.4 تفسیر Error Rate واقعی
بنابراین دو Error Rate باید از هم جدا شوند:

- **Raw JMeter Error Rate = 60.00%**
- **Observed infrastructure/server failure rate (5xx/connection error) = 0% در شواهد ارسال‌شده**

هیچ 500، timeout یا connection failure در اسکرین‌شات‌های ارائه‌شده دیده نمی‌شود.

---

## 5) نتایج واقعی Stress Test – 1000 Request
Stress Run شامل دقیقاً 1000 Sample بوده است.

### 5.1 نتیجه کل Stress

| Metric | مقدار واقعی |
|---|---:|
| Total Samples | **1000** |
| Average | **0 ms** (گرد شده) |
| Median | **0 ms** |
| P90 | **1 ms** |
| P95 | **1 ms** |
| P99 | **1 ms** |
| Min | **0 ms** |
| Max | **2 ms** |
| Std. Deviation | **0.49 ms** |
| Raw JMeter Error % | **60.00%** |
| Throughput | **20.3 req/sec** |
| Received | **5.97 KB/sec** |
| Sent | **3.95 KB/sec** |
| Avg. Bytes | **301.6 bytes** |

### 5.2 جزئیات Stress به تفکیک Label

| Label | Samples | Avg | P95 | P99 | Max | Error % | Throughput |
|---|---:|---:|---:|---:|---:|---:|---:|
| 01 Register | **200** | **0 ms** | **1 ms** | **1 ms** | **2 ms** | **0.00%** | **4.3/sec** |
| 02 List Tickets | **200** | **0 ms** | **1 ms** | **1 ms** | **1 ms** | **0.00%** | **4.3/sec** |
| N1 Invalid tickets user | **200** | **0 ms** | **1 ms** | **1 ms** | **1 ms** | **100.00%*** | **4.3/sec** |
| N2 Invalid ticket reserve | **200** | **0 ms** | **1 ms** | **1 ms** | **1 ms** | **100.00%*** | **4.3/sec** |
| N3 Pay unknown reservation | **200** | **0 ms** | **1 ms** | **1 ms** | **1 ms** | **100.00%*** | **4.3/sec** |

`*` این 100% مربوط به HTTP 4xx مورد انتظار Negative Test است.

### 5.3 تحلیل Stress
در 1000 Request ثبت‌شده:

- P95 و P99 برابر **1 ms** باقی مانده‌اند.
- Max فقط **2 ms** بوده است.
- Register و List Tickets در هر 200 Sample خطای 0% داشته‌اند.
- هیچ شاهدی از 5xx یا connection failure در گزارش ارائه‌شده وجود ندارد.
- Throughput کل به **20.3 request/sec** رسیده است.

از دید latency و stability، اجرای 1000 Request هیچ نشانه‌ای از saturation، tail-latency شدید، crash یا degradation قابل مشاهده نشان نمی‌دهد.

---

## 6) تحلیل P95 / P99
P95 یعنی 95 درصد Requestها در زمانی کمتر یا مساوی مقدار گزارش‌شده پاسخ گرفته‌اند. P99 نیز وضعیت 99 درصد Requestها را نشان می‌دهد و برای تشخیص Tail Latency اهمیت بیشتری دارد.

### Load Run
- P95 = **1 ms**
- P99 = **2 ms**
- Max = **2 ms**

اختلاف P99 با P95 فقط 1ms است؛ در نتیجه Tail Latency قابل توجهی در Run مشاهده نمی‌شود.

### Stress Run
- P95 = **1 ms**
- P99 = **1 ms**
- Max = **2 ms**

در Stress نیز P99 از P95 واگرا نشده است. بنابراین از داده موجود، صف‌شدن شدید، contention زمانی یا saturation در لایه HTTP مشاهده نمی‌شود.

> Averageهای 0ms به معنای «صفر بودن واقعی زمان پردازش» نیستند. چون سرویس روی localhost اجرا شده و JMeter زمان را با دقت میلی‌ثانیه نمایش می‌دهد، زمان‌های کمتر از 1ms می‌توانند به شکل 0ms گرد شوند.

---

## 7) تحلیل Bottleneck
### 7.1 Bottleneck عملکردی مشاهده‌شده
از روی داده واقعی، Bottleneck از جنس CPU/Latency یا Network دیده نمی‌شود. Response Timeها در هر دو Run در بازه 0 تا 2ms باقی مانده‌اند و P99 حداکثر 2ms است.

### 7.2 Bottleneck منطقی/Stateful
مهم‌ترین محدودیت معماری سرویس، State مشترک و ظرفیت محدود بلیت‌هاست. در بار هم‌زمان، کاربران برای Resource محدود رقابت می‌کنند. این موضوع می‌تواند باعث موارد زیر شود:

- نبود ticket آزاد برای ادامه Happy Path؛
- 409 در رزرو روی ticketی که قبلاً رزرو/پرداخت شده است؛
- عدم امکان Pay/Cancel روی Reservation در State نامعتبر؛
- تفاوت نتیجه Runها در صورت Restart نشدن سرویس.

در واقع Bottleneck اصلی این Challenge بیشتر **Business State / inventory contention** است تا latency زیرساخت.

### 7.3 نکته مهم درباره اجرای فعلی
در Aggregate/Summary ارسالی برای Runهای نهایی، Sample مستقلی با Labelهای `03 Reserve`، `04A Cancel` و `04B Pay` مشاهده نمی‌شود. این یعنی Happy Path رزرو/پرداخت/لغو در شواهد فعلی ثبت نشده یا به دلیل نبود ticket آزاد توسط شرط سناریو Skip شده است.

این رفتار با Stateful بودن سرویس سازگار است، مخصوصاً اگر Run بعد از مصرف State قبلی اجرا شده باشد. با این حال، برای اثبات کامل Requirement مربوط به سناریوی `register → tickets → reserve → pay/cancel` بهتر است یک Run تازه بلافاصله پس از Restart سرویس انجام و خروجی Reserve/Pay/Cancel نیز ضمیمه شود.

---

## 8) تحلیل Scalability
داده موجود نشان می‌دهد مجموع پنج Stage با 260 Thread اجرا شده است؛ اما چون Labelهای یکسان بین Stageها در Listener تجمیع شده‌اند، اسکرین‌شات فعلی فقط نتیجه Aggregate کل Stageهای 10، 30، 50، 70 و 100 را نشان می‌دهد و P95/P99 مستقل هر Stage از روی این تصاویر قابل بازیابی نیست.

با این وجود، خروجی Aggregate کل بسیار پایدار است:

- Register: P95/P99 = **2/2 ms**
- Tickets: P95/P99 = **1/1 ms**
- Total: P95/P99 = **1/2 ms**
- Maximum = **2 ms**

بنابراین هیچ علامت آشکاری از افت عملکرد تا بار تجمیعی اجراشده مشاهده نشده است.

برای گزارش Trend دقیق Stage-by-Stage در Run بعدی، بهتر است Label هر Stage متفاوت باشد (مثلاً `Load10-Register`, `Load30-Register`, ...) یا خروجی JTL بر اساس `threadName` فیلتر شود.

---

## 9) تحلیل خطاها
### 9.1 4xx
4xxهای مشاهده‌شده بخشی از Negative Testing هستند. مهم‌ترین موارد:

- 401 برای userId نامعتبر
- 404 برای ticketId نامعتبر
- 404 برای reservationId نامعتبر

این پاسخ‌ها نشان می‌دهند API ورودی/State نامعتبر را با پاسخ معنادار Reject کرده است.

### 9.2 5xx
در هیچ‌یک از Summary/Aggregate Reportهای ارسال‌شده خطای 500 یا سایر 5xxها مشاهده نشد.

**Observed 5xx rate = 0%**

### 9.3 Connection Error / Timeout
در شواهد ارسالی connection failure یا timeout مشاهده نشده است.

**Observed connection error rate = 0%**

### 9.4 نتیجه خطا
بنابراین Error% خام 60% نباید به عنوان Failure Rate سیستم گزارش شود؛ این 60% عملاً از Negative Pathهای 401/404 تشکیل شده و رفتار مورد انتظار Contract است.

---

## 10) بررسی Memory Leak و پایداری
JMeter می‌تواند latency، throughput، response code و خطاهای ارتباطی را ثبت کند، اما از اسکرین‌شات‌های Summary/Aggregate به تنهایی نمی‌توان وجود یا عدم وجود memory leak در Process Node.js را اثبات کرد.

در اجرای فعلی:

- Process تا پایان Stress Run پایدار مانده است.
- 1000 Request کامل شده‌اند.
- هیچ crash یا connection error در شواهد وجود ندارد.
- هیچ افزایش latency غیرعادی مشاهده نشده است.

پس **نشانه عملی از ناپایداری در این Run دیده نشده**، اما برای ادعای قطعی «عدم memory leak» باید RSS/Heap Node.js در ابتدای تست، Peak و چند دقیقه بعد از پایان تست ثبت شود.

ریسک بالقوه معماری این است که `users` و `reservations` در حافظه Process نگه‌داری می‌شوند و در تست‌های طولانی‌تر، بدون cleanup/TTL، رشد Memory ممکن است رخ دهد.

---

## 11) Repeatability و اثر State
این API Stateful است؛ بنابراین Restart نکردن سرویس بین Runها می‌تواند نتیجه را تغییر دهد. Ticket پرداخت‌شده دوباره Available نمی‌شود و User/Reservationهای ایجادشده نیز در Memory باقی می‌مانند.

برای مقایسه علمی Runها:

1. Node.js Restart شود.
2. JMeter Listenerها Clear شوند.
3. Run جدید اجرا شود.
4. `results.jtl` جداگانه ذخیره شود.
5. فقط Runهایی با State اولیه یکسان با هم مقایسه شوند.

---

## 12) جمع‌بندی نهایی
بر اساس داده‌های واقعی ارائه‌شده:

- Load/Scalability Run شامل **1300 Sample** بوده است.
- Stress Run شامل **1000 Sample** بوده است.
- در Load Run، P95 = **1ms** و P99 = **2ms** ثبت شده است.
- در Stress Run، P95 = **1ms** و P99 = **1ms** ثبت شده است.
- Max Response Time در هر دو Run فقط **2ms** بوده است.
- Throughput کل Load Run برابر **8.9 req/sec** و Stress Run برابر **20.3 req/sec** بوده است.
- Register و List Tickets در هر دو Run **0% Error** داشته‌اند.
- Raw Error% برابر **60%** است، اما این مقدار از Negative Testهای 401/404 تشکیل شده و نشانه Failure زیرساخت نیست.
- **هیچ 5xx، timeout یا connection failure در شواهد ارسالی مشاهده نشده است.**
- هیچ نشانه‌ای از افزایش شدید P95/P99، Tail Latency یا saturation دیده نشد.
- Bottleneck محتمل سرویس بیشتر از جنس **State مشترک، محدودیت inventory و contention روی تعداد محدود Ticketها** است، نه توان پاسخ‌گویی HTTP.
- از نظر Stability، اجرای 1000 Request بدون crash و بدون 5xx/connection failure موفق بوده است.
- درباره memory leak، داده فعلی نشانه‌ای از failure نشان نمی‌دهد ولی برای اثبات قطعی باید RSS/Heap Process جداگانه مانیتور شود.

### نتیجه نهایی ارزیابی
با شواهد فعلی، سرویس در بار آزمایش‌شده از نظر latency و پایداری HTTP رفتار بسیار مناسبی داشته است و حتی در Stress 1000 Request نیز P99 در 1ms باقی مانده است. مهم‌ترین ریسک سیستم به جای Performance خام، مدیریت State و ظرفیت محدود Resourceهاست. پاسخ‌های 4xx مشاهده‌شده عمدتاً رفتار عمدی و صحیح Negative Test بوده‌اند و هیچ نشانه‌ای از خطای 5xx یا شکست ارتباطی دیده نشده است.

---

## 13) محدودیت شواهد و پیشنهاد برای نمره کامل
برای کامل‌تر کردن مستندات و حذف هرگونه ابهام در ارزیابی نهایی، این موارد در یک Run تازه توصیه می‌شود:

- Restart سرویس قبل از Run تا ticketها Available باشند.
- ثبت Sampleهای `03 Reserve`, `04A Cancel`, `04B Pay` در Happy Path.
- ذخیره فایل واقعی `results.jtl` و HTML Dashboard.
- گرفتن نمودارهای `Response Times Over Time`, `Percentiles`, `Active Threads Over Time`, `Errors`.
- ثبت RSS/Heap Node.js قبل، حین Peak و پس از تست برای بررسی Memory Leak.
- جدا کردن Metricهای Stageهای 10/30/50/70/100 با Label یا threadName مجزا تا روند Scalability به‌صورت دقیق Stage-by-Stage قابل گزارش باشد.

این موارد داده‌های فعلی را نقض نمی‌کنند؛ فقط شواهد را برای Requirementهای کامل Challenge قوی‌تر می‌کنند.
