---
title: "How to count occurences in Bigquery Array/Repeated Fields"
date: 2021-11-11T21:01:23Z
draft: true
tags: ["bigquery", "sql", "tips"]
---

In BigQuery, there is the concept of [repeated fields and arrays](https://medium.com/google-cloud/bigquery-explained-working-with-joins-nested-repeated-data-1941646ccb5b). I was trying to figure out how to count how many entries in a table contain a certain value in that repeated column. And since the syntax `[value] IN some_array` is not valid, there is one or two steps extra that is needed.

Suppose we have some data around freemium podcast platform. In this platform users can pay for an upgraded service, or listen for free. The users can pay in multiple different ways (e.g. they might subscribe to only certain podcasts, or pay for a 'everything' service). The service also hold regular events to tempt users to subscribe. Our table has these fields


, we hold the data for their subscription info in an array, e.g. 



