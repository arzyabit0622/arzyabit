# گزارش عملکرد پیک‌ها

فرض: چون Schema کامل ارائه نشده، برای فیلتر زمانی ستون `created_at` در جدول `deliveries` در نظر گرفته شده است و مقدار `status` برای ارسال‌های نهایی `success` یا `failed` است.

## 1. تعداد سفر موفق و ناموفق به تفکیک پیک و روز

```sql
SELECT
    courier_id,
    DATE(created_at) AS delivery_day,
    SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_trips,
    SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) AS failed_trips,
    COUNT(*) AS total_trips
FROM deliveries
WHERE created_at >= :from_date
  AND created_at < :to_date
  AND status IN ('success', 'failed')
GROUP BY courier_id, DATE(created_at)
ORDER BY delivery_day, courier_id;
```

نمونه خروجی فرضی:

| courier_id | delivery_day | successful_trips | failed_trips | total_trips |
|---:|---|---:|---:|---:|
| 101 | 2026-09-10 | 12 | 2 | 14 |
| 102 | 2026-09-10 | 9 | 1 | 10 |
| 101 | 2026-09-11 | 11 | 3 | 14 |

## 2. همان گزارش روزانه همراه با نرخ موفقیت کل پیک در بازه

```sql
SELECT
    daily.courier_id,
    daily.delivery_day,
    daily.successful_trips,
    daily.failed_trips,
    totals.total_trips_in_range,
    ROUND(
        100.0 * totals.successful_trips_in_range /
        NULLIF(totals.total_trips_in_range, 0),
        2
    ) AS success_rate_in_range
FROM (
    SELECT
        courier_id,
        DATE(created_at) AS delivery_day,
        SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_trips,
        SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) AS failed_trips
    FROM deliveries
    WHERE created_at >= :from_date
      AND created_at < :to_date
      AND status IN ('success', 'failed')
    GROUP BY courier_id, DATE(created_at)
) AS daily
JOIN (
    SELECT
        courier_id,
        SUM(CASE WHEN status = 'success' THEN 1 ELSE 0 END) AS successful_trips_in_range,
        COUNT(*) AS total_trips_in_range
    FROM deliveries
    WHERE created_at >= :from_date
      AND created_at < :to_date
      AND status IN ('success', 'failed')
    GROUP BY courier_id
) AS totals
    ON totals.courier_id = daily.courier_id
ORDER BY daily.delivery_day, daily.courier_id;
```

نمونه خروجی فرضی:

| courier_id | delivery_day | successful_trips | failed_trips | total_trips_in_range | success_rate_in_range |
|---:|---|---:|---:|---:|---:|
| 101 | 2026-09-10 | 12 | 2 | 28 | 82.14 |
| 102 | 2026-09-10 | 9 | 1 | 21 | 90.48 |
| 101 | 2026-09-11 | 11 | 3 | 28 | 82.14 |
