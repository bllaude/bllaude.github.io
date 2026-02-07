---
layout: post
title:  Fibonacci
date:   2020-06-13 22:08:00 +0800
categories: document
tag: programming
topping: false
---

* content
{:toc}
I was reading SICP recently and it was a lot of pressure to let go of my old worldview, but totally worth it. For example, I learned the `0(log(n))` Fibonacci algorithm and got a big posture.

I remember a written test I saw a few days ago, saying that it was the fastest way to calculate the `nth` term of the Fibonacci sequence. Although I knew there was a general term formula, it wasn't suitable for accurate calculations, so I wrote an iterative algorithm. At the time, I thought it was great, now it feels too simple, even a bit naive.

Everybody knows that the definition of Fibonacci is `f(n) = f(n-1) + f(n-2)`; `f(1) = f(2) = 1` (counting from `f(1)=1`.)

Python version:

```python
fibonacci =lambda n: 1 if n <= 2 elsefibonacci(n-1) + fibonacci(n-2)
/* C version*/
int fibonacci(int n) {
    returnn <= 2? 1: fibonacci(n-1) + fibonacci(n-2);
}

; Scheme version
(define (fibonacci n) (if (<= n 2) 1 (+ (fibonacci (- n 1)) (fibonacci (- n 2)))))
Unfortunately, the efficiency of tree expansion is too low. When n=42, the C language needs more than 1s.

Therefore, it needs to be changed to an iterative version, using (a, b) <- (a + b, a) this way. #Python Edition

deffibonacci(n):
    a = 1; b = 0;
    for i inrange(n):
        a, b = a + b, a
    return
```

C version:
```c
int fibonacci(int n) {
    int a = 1, b = 0, c;
    while(n--) {
        c = a;
        a = a + b;
        b = c;
    }
    returnb;
}

;Scheme version
(define(fibonacci n)
  (define(iter nab)
    (if(= n 0) b (iter (- n 1) (+ ab) a)))
  (iter n 1 0))
```

Although the efficiency of `O(n)` has been significantly improved, since the growth of this sequence exceeds `2^n`, for a slightly larger `n`, large integer operations are required, which is very inefficient. For the python version of the code, when `n=300,000`, it takes more than 1s to get the result; for the Scheme version, it takes about 9s.

SICP exercises 1-19 mentioned an ingenious `O(log(n))` algorithm: each iteration of calculating fibonacci `(a, b) <- (a + b, a)` is expressed as a transformation `T[p =0, q=1]`, specifically expressed as (it seems to be reversed by matrix multiplication) `Tpq = (a(p+q) + bq, aq + bp)`.

By calculating `Tpq` (that is, performing `T[pq]` transformation on `(a, b)` twice), we can get `(a((pp+qq) + (2pq+qq)) + b(2pq+qq), a(2pq+qq) + b(pp+qq))`.

denote `p’= pp + qq, q’= 2pq + qq` has `Tp’q’ = Tpq = (a(p’ + q’) + bq’, aq’ + bp’)`.

So `f(n+1) = T[pq] n (f(1))`

That is to say — fibonacci can be calculated by an algorithm similar to divide and conquer *“a to the nth modulus b”*!

After the algorithm is given, the code is easier to write:

Python version:
```python
def fibonacci(n):
    def iter(a, b, p, q, n):
        if n == 0:
            return b
        elif n% 2 == 0:
            return iter(a, b, p * p + q * q, 2 * p * q + q * q, n / 2)
        else:
            return iter(a * (p + q) + b * q, a * q + b * p, p, q, n-1)
    returniter(1, 0, 0, 1, n)
```

C version:
```c
typedef unsigned long longull;
ull fibo_iter(ull a, ull b, ull p, ull q,int n) {
    if (n == 0)
        return b;
    else if (n% 2 == 0)
        return fibo_iter(a, b, p *p + q * q, 2 * p * q + q * q, n / 2);
    else
        returnfibo_iter(a * (p + q) + b * q, a * q + b * p, p, q, n-1);
}

ull fibonacci(int n) {
    returnfibo_iter(1, 0, 0, 1, n);
}

/* Scheme version*/
(define(even? n) (= (remainder n 2) 0))
(define(fibonacci n)
    (define(iter abpqn)
        (cond ((= n 0) b)
              ((even? n)
                (iter
                    a
                    b
                    (+ (* pp) (* qq))
                    (+ (* 2 pq) (* qq))
                    (/ n 2)))
              (else
                (iter
                    (+ (* a (+ pq)) (* bq))
                    (+ (* aq) (* bp))
                    p
                    q
                    (- n 1)))))
    (iter 1 0 0 1 n))
```

After optimization, the efficiency is remarkable. For `n=300,000`, the python version only needs less than 0.04s, and the Scheme version only needs 0.12s to get the result.
