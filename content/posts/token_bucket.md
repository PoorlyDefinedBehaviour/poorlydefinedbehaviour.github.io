---
title: "Token bucket"
date: 2022-03-20T22:21:07-03:00
categories: ["today-i-learned", "algorithms", "rate-limiting"]
draft: true
---

# Intro

[Token bucket](https://en.wikipedia.org/wiki/Token_bucket) is an algorithm that can be used to rate limit requests received by a service.

# How it works

The algorithm is called token bucket because of they way it works: imagine we have a bucket with `x` tokens where each accepted request consumes one token from the bucket and a token is added back to the bucket at an interval.

A bucket with `1` token that is filled each second means the service accepts one request per second.

A bucket with `5` tokens that are refilled every `1/5` seconds means the service accepts `5` requests per second.
