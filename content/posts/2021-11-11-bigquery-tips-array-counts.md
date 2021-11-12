---
title: "How to count occurences in Bigquery Array/Repeated Fields"
date: 2021-11-11T21:01:23Z
draft: false
tags: ["bigquery", "sql", "cheatsheet"]
---

In BigQuery, there is the concept of [repeated fields and arrays](https://medium.com/google-cloud/bigquery-explained-working-with-joins-nested-repeated-data-1941646ccb5b). I was trying to figure out how to count how many entries in a table contain a certain value in that repeated column. And since the syntax `[value] IN some_array` is not valid, there is one extra that is needed.


Suppose we have some data around freemium podcast platform. In this platform users can pay for an upgraded service, or listen for free. The users can pay in multiple different ways (e.g. they might subscribe to only certain podcasts, or pay for a 'everything' service). In this first scenario, we have a `Users` table, with various information about them, the most relevant being `user_id` (of type string) and `subscriptions` (repeated array of strings corresponding to names of podcasts or `unlimited`), and we would like to count users who can access the subscribers only content on the podcasts `science weekly` and `magic101`.

This can be done as follows:


```sql
WITH EligibilityCounts AS (
    SELECT
        user_id,
        COUNTIF( sub IN ('science weekly', 'magic101', 'unlimited')) AS paying_member,
    FROM
        `Users` u
    LEFT JOIN 
        UNNEST(u.subscriptions) sub
    GROUP BY
        user_id
)
SELECT * 
FROM 
   EligibilityCounts
WHERE  
    paying_member > 0
```



