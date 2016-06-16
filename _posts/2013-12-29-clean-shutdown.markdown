---
layout: post
title: "clean shutdown"
date: 2013-12-29 11:53:31 +0000
comments: true
categories: 
---

If you start goroutines in your code, you should stop them, too. If you don't your benchmarks end up being erratic, your tests fail intermittently, and composing your code to run multiple services in one test becomes confusing.

If you've got some non-trivial Go code, try sending SIGQUIT to the process and look at the stacktrace. If you started your process from Bash, you can send it SIGQUIT by hitting Ctrl-&#92;. Ideally, you should be able to account for every goroutine listed in the stacktrace, even if they're started by the runtime.
