# چالش 2 - تحلیل Trace و Metrics

## log.json

همه رویدادها `trace_id = TR-20230423-9991` دارند، بنابراین مسیر یک درخواست از ثبت‌نام تا پرداخت و بررسی مدارک قابل اتصال است.

| زمان | service / event | فاصله با event قبلی |
|---|---|---:|
| 09:45:01 | registration / request_received | شروع |
| 09:45:02 | registration / validation_passed | 1s |
| 09:45:03 | payment / payment_initiated | 1s |
| 09:45:04 | payment / payment_processing | 1s |
| 09:45:09 | payment / payment_success | 5s |
| 09:45:10 | document / doc_submission | 1s |
| 09:45:12 | document / doc_verification_started | 2s |
| 09:46:30 | document / doc_verification_failed | 78s |

کل مسیر از `request_received` تا شکست مدارک حدود 89 ثانیه است. بیشترین توقف Trace بعد از `doc_verification_started` رخ داده و 78 ثانیه تا خطا فاصله دارد.

## metrics.md

| service | نتیجه | avg_response_time | برداشت |
|---|---|---:|---|
| registration | 1 موفق، 0 خطا | 200ms | رفتار عادی در این نمونه |
| payment | 1 موفق، 0 خطا | 6000ms | کندتر از registration ولی موفق |
| document | 0 موفق، 1 خطا | 80000ms | بیشترین latency و تنها خطای مسیر |

Metric سرویس document با Trace هم‌راستا است: از `doc_submission` در 09:45:10 تا `doc_verification_failed` در 09:46:30 دقیقاً 80 ثانیه طول کشیده است.

## flow.drawio

وابستگی مسیر این است که بررسی مدارک بعد از پرداخت موفق شروع می‌شود. بنابراین payment با وجود latency حدود 6 ثانیه، علت شکست نهایی این Trace نیست. نقطه بحرانی در مرحله بررسی مدارک قرار دارد.

نسخه Annotated در `analysis.drawio` همین مسیر را با زمان‌ها و نقطه گلوگاه مشخص می‌کند.

## منشأ اختلال

منشأ قابل اثبات اختلال، مرحله `doc_verification` در سرویس document است:

- Trace بعد از شروع بررسی مدارک 78 ثانیه بدون رویداد بعدی می‌ماند و سپس شکست ثبت می‌شود
- Metric سرویس document برابر 80000ms است که با همین فاصله زمانی همخوانی دارد
- document در نمونه 1 درخواست، 1 خطا و 0 موفق دارد
- registration و payment هر دو قبل از آن موفق شده‌اند

ریشه فنی دقیق داخل document service از داده فعلی قابل اثبات نیست. در لاگ خطای `doc_verification_failed`، `error_code`، `error_message`، dependency مقصد، `span_id`، تعداد retry و timeout ثبت نشده است. بنابراین نسبت دادن مشکل به DB، سرویس بیرونی یا Queue بدون شواهد درست نیست.

برای بستن Root Cause در رخداد بعدی، روی مرحله verification این اطلاعات اضافه شود:

- `span_id` و نام dependency برای هر call داخلی/خارجی
- `duration_ms` برای هر dependency call
- `error_code` و `error_message`
- `attempt/retry_count` و ثبت timeout در صورت وقوع
- Metricهای `verification_latency_p95`، `verification_error_rate`، `timeout_count`، `retry_rate` و در صورت وجود صف `queue_depth`

پرداخت 6000ms هم باید جداگانه با baseline/SLO بررسی شود، ولی در این Trace عامل شکست فرآیند نیست.
