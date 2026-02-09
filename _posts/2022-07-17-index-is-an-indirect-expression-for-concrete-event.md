---
layout: post
title:  Index is an indirect expression for concrete event
date:   2022-07-17 22:08:00 +0800
categories: document
tag: programming
topping: false
---

* content
{:toc}
I was put in charge of improving the quality of an appl in a particular project, so I decided to use indicators to quantitatively express the quality of the software.

This is because various indicators are sometimes used to prove the quality, especially in projects that have been ordered by customers.

- Number of tests
- Number of reviews
- Test density (= number of tests / number of lines of code
- Bug density (= number of bugs found / number of tests)
- Code coverage
- Number of lint warnings

… are commonly used indicators of software quality.

By using these indicators, the quality of the software is analyzed, and it is determined that there are no quality problems, and the release is approved.

However, if we recall the definition of an index (calculation method) again, we can see that an index is something that expresses a certain phenomenon numerically and is used to objectively grasp the situation.

In other words, the index does not directly indicate each specific event itself . It’s just an indirect expression .

That’s not to say that the above metrics in software development don’t represent quality well. Rather, it is a wonderful tool that can be of great use in analyzing quality from various angles.

**What I want to say is that all indicators within target range ≠ good quality.**

For example, if we consider the number of lint warnings, it indicates the number of places where the writing style that is defined as a problem in the linter is used, but conversely, any writing style that is not defined as a problem is allowed. . Therefore, it is possible to intentionally avoid the warning in a strange way of writing. In that case, it’s usually the worst code with low readability.

Code coverage is an indicator that the code has been tested and its functionality has been proven. The risk of containing bugs cannot be denied.

In this way, even if the index itself is considered to be no problem, there are cases where the quality is not as indicated by the index when looking at the actual situation inside.

Even so, it is impossible to look at every specific event in detail, so we try to grasp the overall picture using indicators, and if we are in a position to supervise the project, it is natural. That’s the thing.

That’s why the site must do its best to keep the specific events that formed the index as it should be.
