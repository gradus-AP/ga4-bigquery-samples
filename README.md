# ga4-bigquery-samples

## Overview

BigQueryでGA4のイベント集計を行う際に使えるクエリの例。

## イベント集計

各種イベント数の集計は、イベントに相当するレコードの行数をカウントすることで行うことができます。

**1. `COUNTIF`関数を使う**

```sql
SELECT
    COUNT(DISTINCT user_pseudo_id) AS users,
    COUNTIF(event_name = 'session_start') as sessions,
    COUNTIF(event_name = 'page_view') as page_views
FROM
    `{project name}.analytics_{ID}.events_*`
WHERE
    _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
```

**2. `GROUP BY`句を使う**

```sql
SELECT
    event_name,
    count(*) as events
FROM
  `{project name}.analytics_{ID}.events_*`
WHERE
    event_name IN ('page_view', 'session_start')
    AND _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
GROUP BY event_name
```

## `event_params`列の中身を列に展開する

`event_params`列は繰り返しレコード型になっているため、集計時には中身を展開(フラット化)する必要があります。<br/>
[Converting elements in an array to rows in a table](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays?hl=en#flattening_arrays)

**`UNNEST`演算子を使って繰り返し列をフラット化**

```sql
/**
 * イベントテーブルにevent_paramsの中身のkey, valueを展開した列をつける
 */
SELECT
    key,
    value.string_value,
    value.int_value,
    value.float_value,
    value.double_value
FROM
    `{project name}.analytics_{ID}.events_*`
    CROSS JOIN UNNEST(event_params)
WHERE
    _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
```

**ページURLごとにユーザー数、セッション数、PV数を集計する**

```sql
SELECT
    value.string_value AS page_URL,
    count(DISTINCT user_pseudo_id) AS users,
    countif(event_name = 'session_start') as sessions,
    countif(event_name = 'page_view') as page_views
FROM
    `{project name}.analytics_{ID}.events_*`
    CROSS JOIN UNNEST(event_params)
WHERE
    key = 'page_location' -- ページURL
    AND _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
GROUP BY
    value.string_value
```

## `WITH`句

`WITH`句を使うと、サブクエリに対して名前をつけて参照できます。セグメント化集計のような、クエリが長く複雑になりがちな場合に有効です。

**初回来訪とリピーターにセグメント分けを行い、ユーザー数、セッション数、PV数を集計**

```sql
/**
 * 初回来訪/リピーターでセグメント分け(初回来訪(is_new_user = 1)/
 * リピーター(is_new_user = 0))し、UserInfoと名前をつけ、イベントテーブルと
 * 左外部結合を行う。
 */
WITH 
UserInfo AS (
    SELECT
        user_pseudo_id,
        MAX(IF(event_name = 'first_visit', 1, 0)) AS is_new_user
    FROM
        `{project name}.analytics_{ID}.events_*`
    WHERE
        _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
    GROUP BY user_pseudo_id)
SELECT
    UserInfo.is_new_user,
    COUNT(DISTINCT UserInfo.user_pseudo_id) AS user_count,
    COUNTIF(event_name = 'session_start') AS sessions,
    COUNTIF(event_name = 'page_view') AS page_views
FROM 
    UserInfo
    LEFT JOIN (
    SELECT
        user_pseudo_id,
        event_name
    FROM
        `{project name}.analytics_{ID}.events_*`
    WHERE
        _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
    ) AS T
    ON UserInfo.user_pseudo_id = T.user_pseudo_id
GROUP BY
    is_new_user
```

## ユーザー単位の集計、セッション単位の集計

ユーザー単位の集計を行いたい場合、user_pseudo_id でグループ化を行います。

**ユーザーを最初に獲得した流入元、メディア別のセッション数、PV数をユーザー単位で集計**

```sql
/**
 * ユーザーを最初に獲得した流入元、メディア別のセッション数、PV数をユーザー単位で集計
 * ユーザー単位の集計 -> user_pseudo_idでグループ化
 */
SELECT
    user_pseudo_id,
    traffic_source.source, 
    traffic_source.medium,
    COUNTIF(event_name = 'page_view') AS page_views,
    COUNTIF(event_name = 'session_start') AS sessions
FROM
    `{project name}.analytics_{ID}.events_*`
WHERE
    _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
GROUP BY user_pseudo_id, traffic_source.source, traffic_source.medium
```

セッション単位で集計を行いたい場合、`user_pseudo_id`と`ga_session_id`の両方でグループ化を行います。<br/>
`ga_session_id`は各イベントの`event_params`列に格納されているので、展開して取り出すことで集計できます。

**デバイスカテゴリ×各セッションごとにPV数を集計**


```sql
/**
 * デバイスカテゴリ(desktop, mobile, tablet)ごとに各セッション
 * のPV数を取得する
 */
SELECT
    user_pseudo_id,
    value.int_value as ga_session_id,
    device.category as device_category,
    COUNTIF(event_name = 'page_view') as page_views
FROM
    `{project name}.analytics_{ID}.events_*`
    CROSS JOIN
    UNNEST(event_params)
WHERE
    key = 'ga_session_id'
    AND _TABLE_SUFFIX BETWEEN '20221101' AND '20221130'
GROUP BY
    user_pseudo_id, value.int_value, device.category
```

## References

- [[GA4] BigQuery Export スキーマ](https://support.google.com/analytics/answer/7029846?hl=ja)
- [Google アナリティクス 4 のイベントデータのエクスポートに関する基本的なクエリ](https://developers.google.com/analytics/bigquery/basic-queries)
- [GA4用のBigQuery クエリ集(GA4ガイド)](https://www.ga4.guide/related-service/big-query/query-writing/)