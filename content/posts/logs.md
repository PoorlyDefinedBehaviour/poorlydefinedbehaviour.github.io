---
title: "Logs"
date: 2022-04-30T16:46:09-03:00
categories: ["algorithms", "distributed", "data-structures"]
draft: true
---

# What is a log

A log is just a immutable sequence of records. It is usually 0 indexed and grows from left to right.

<p align="center">
<img src="https://user-images.githubusercontent.com/17282221/168452116-a751154f-ec58-4a65-91f5-a90269529963.png" />
</p>

> Not to be confused with the type of logs most people are used to: application logs that are meant to be read by humans.

What i called a record is a entry in the log. The entry can be anything in any format.

# You have seen a log before

Logs are used everywhere, from databases, most file systems and distributed systems.
