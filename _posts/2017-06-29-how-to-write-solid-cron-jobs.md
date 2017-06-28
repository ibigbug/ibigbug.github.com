---
layout: post
title: How to write solid cron jobs
date: 2017-06-29
category: work
meta: 6 tips for writing cron jobs
---

Recently I migrate [pypi-mirrors](https://pypi-mirrors.org) from [Vultr](http://www.vultr.com/?ref=7028689) VPS to Racher stack, which is a pure container environment. Everything work fine, though it took me some time to setup the cron job in container correctly. And here is a short summary which might need keep in mind while using cron job inside a container.

### 1. Environment Variables

Since docker containers variables were set via parameters during the container start. It's not very easy be accessed from the cron job, because the environment variables are always inside the process whose pid is 1 and can not be read outside the process directly.

The work around is exporting the environment variables to a file during the [entrypoint](https://github.com/ibigbug/pypi-mirrors/blob/76df880ba5510e463eb1c86c6adf10295c9f17c0/scripts/entrypoint#L6) run and then [source](https://github.com/ibigbug/pypi-mirrors/blob/76df880ba5510e463eb1c86c6adf10295c9f17c0/crontab#L2) the file before the real cron job command run.

### 2. Use absolute path as much as possible

For example, prefer to use `/usr/local/bin/python` rather than `python` as much as possible. In most cases, this will save you time to figure out the `$PATH` things.

### 3. Do necessary logging

Output useful information to stdout/stderr, this will help you when bad thing happens.

### 4. Propery redirect

In most cases, using something like `> /var/log/xxx.log 2>&1` would help you observe if your cron jobs are running as expected.

### 5. Use `.` instead of `source`

Since `/bin/sh` is not always equal to `/bin/bash`, using `.` to do a source is much safer.

### 6. Debug

If you really not sure wether your cron job is running, try to install `syslog` or `rsyslog`, then uncomment the `cron` related logging. Then you'll see something in `/var/log/cron.log`, which might be useful.
