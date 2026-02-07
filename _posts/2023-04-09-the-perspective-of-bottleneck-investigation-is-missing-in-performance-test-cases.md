---
layout: post
title:  The perspective of bottleneck investigation is missing in performance test cases
date:   2023-04-09 22:08:00 +0800
categories: document
tag: notes
topping: false
---

* content
{:toc}
It goes without saying that test cases are necessary to conduct performance tests, but in most cases there is a lack of perspective to be confirmed when creating the cases.

There’s no doubt that the purpose of performance testing is to *“confirm whether the required performance is satisfied”*, but this is not essential. The essence of performance testing should be to "*find and correct the parts that do not meet the required performance, so that it can be met”*.

Because this way of thinking is missing, there's a lack of metrics acquisition for bottleneck investigation from the perspective of test case confirmation.

Performance tests often have a very high cost per execution, so if you find that the required performance is not met and you notice that the metrics are not enough, repeat the high cost case. It has to be done and schedule delays are inevitable.

The solution to this problem is simply careful case building.

By carefully considering the architecture used in the case and its processing content, and predicting where bottlenecks are likely to occur, we will come up with the metrics that should be obtained. Here are some places to focus on:

- If you are using externally provided APIs or cloud services, is there a usage limit? If you’re using queues, what are their throughput limits?
- When DB processing is included, the upper limit of the number of connections is some
- What is the storage capacity and speed of access to storage if temporary storage is included?

In many cases, if you look at it from this point of view, you will be able to identify the metrics that you should obtain and the areas that are likely to be bottlenecks.

However, even if it is for investigation, be careful not to output too many logs. In some cases, log output may affect performance, so try to keep the amount of log to a minimum even if you output it in multiple places.
