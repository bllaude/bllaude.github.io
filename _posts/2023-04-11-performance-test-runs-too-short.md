---
layout: post
title:  Performance test runs too short
date:   2023-04-11 22:08:00 +0800
categories: document
tag: notes
topping: false
---

* content
{:toc}
You probably notice that performance testing costs a lot of time to run one case. Moreover, it carries the risk of failing the execution itself. Furthermore, fixes to eliminate bottlenecks found in performance testing are often far more expensive than functional fixes.

I don’t know if it’s because there are few people who know this reality, or because the emphasis is on functionality, but in most cases, the performance test period is quite short.

The solution to this problem, of course, is a long performance test period.

You might hear people say, *“That’s difficult!”* That is why I'd like you to use the point of view that were originally not sufficient for conducting performance tests, as mentioned above, as the basis for estimates to ensure a sufficient period for performance tests.

Specifically, it looks like this:

- For each case with different characteristics, one trial case is prepared, and the execution time and the time for correcting defects discovered by it are allocated. This enables subsequent performance test execution to be performed smoothly and reduces the risk of failure of the performance test execution itself.
- Approximately XXX metrics and logs are required to verify the performance test results, so the environment preparation period for the performance test is required.
- This period is necessary because it is necessary not only to execute the performance test but also to carefully verify the results.
- As a result of verifying the results, problems are found, and most of the time, the correction is more complicated than the functional test, so the period for correction is XX% longer than the functional test.
- In particular, the fourth point should be emphasized. As I wrote in Problem 1, the essence of performance testing lies in *“finding and fixing problems.”* Planning for *“fixing”* is extremely important, but it is a step that is often neglected in any test plan.

By estimating in the performance test plan in this way, it will be easier to secure more performance test periods than before.
