---
layout: post
title:  "Hello World"
date:   2026-01-02 12:00:00 +0000
tags: [networking, datacenter, notes]
---

Hello world! This is my first post on this Jekyll blog.

I'm setting up this site to document my networking notes.

Here is an example of a networking command:

```bash
# Check interface status
ip addr show dev eth0
```

And a snippet of Python for automation:

```python
import os

def check_ping(hostname):
    response = os.system("ping -c 1 " + hostname)
    if response == 0:
        print(f"{hostname} is up!")
    else:
        print(f"{hostname} is down!")

check_ping("8.8.8.8")
```
