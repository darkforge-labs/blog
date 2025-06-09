---
layout: post
title:  "HackCentral CTF"
date:   2025-06-09 10:00:00 +0200
categories: CTF Challenge Giveaway
---

## Introduction

Thank you to everyone who participated in the HackCentral CTF, we hope you enjoyed solving the challenge as much as we enjoyed building it. Congratulations to Calum Rivers for being the first to solve the challenge, and to [@deNable_D](https://x.com/deNable_D) for winning the giveaway! We hope you enjoy your prize.

![Banner for HackCentral](/assets/images/57b1fd957a20ad9b6356541c5ae02921c340e1ed.png)

Inspired by Orange Tsai’s research into Apache confusion attacks, we set out to create a custom CTF challenge that combines the DocumentRoot confusion vector with a lesser-known rewrite rule bypass technique.

The challenge source code is available at:
* [https://github.com/darkforge-labs/ctf-hackcentral](https://github.com/darkforge-labs/ctf-hackcentral)

You can also try the live version at:
* [https://hackcentral.darkfor.ge/](https://hackcentral.darkfor.ge/)

## The Challenge

We begin by examining the [Dockerfile](https://github.com/darkforge-labs/ctf-hackcentral/blob/main/Dockerfile). Nothing appears immediately suspicious, but we note that `start.sh` is executed on container startup. On line 6, this script generates a random 64-character string, writes it to `secret.txt`, and places the file within the webroot at `/var/www/html`.

![start.sh](/assets/images/e38c9ed8d98f62b03c5c13d5788b43a6f39e4580.png)

Within the application files, we find a single script: `getFlag.php`. This script returns the contents of `flag.txt` **only if** the user supplies a valid `session` cookie. Otherwise, it returns a `403 Forbidden`. Therefore, our first task is to gain access to `secret.txt`.

![PHP Contents](/assets/images/80e94a8ff606688d93d5f4a6844d31ac66b6658b.png)

## Accessing secret.txt

Although `secret.txt` is placed in the webroot, requesting it directly [https://hackcentral.darkfor.ge/secret.txt](https://hackcentral.darkfor.ge/secret.txt) returns a 403. This is due to Apache rewrite rules defined in [apache.conf](https://github.com/darkforge-labs/ctf-hackcentral/blob/main/apache.conf), where both direct access (`/secret.txt`) and requests via the `/old/` path are denied (see lines 2 and 5).

![apache configuration](/assets/images/c85c6ad2ee8170980ea87f4014a967f0c26baa85.png)

Interestingly, `/old/` is configured to rewrite requests to the root directory. Since the Deny rules block `/secret.txt` and `/old/secret.txt`, we need to find an alternate path that circumvents these restrictions.

## DocumentRoot Confusion

As detailed in Orange Tsai’s post on [DocumentRoot Confusion](https://blog.orange.tw/posts/2024-08-confusion-attacks-en/#%F0%9F%94%A5-2-DocumentRoot-Confusion), Apache may attempt to resolve the request both with and without prepending the DocumentRoot.

This means we can trick Apache into loading a file using its full path from within a subdirectory. A request to:
* `https://hackcentral.darkfor.ge/old/var/www/html/secret.txt`
bypasses the rewrite rules and successfully discloses the contents of `secret.txt`.
![secret.txt](/assets/images/3cb522c765d251d5b50d6720942aa87ee9235250.png)

## Accessing getFlag.php

With the value of `secret.txt` in hand, we are ready to retrieve the flag by making a request to `getFlag.php` with the `session` cookie set. However, `getFlag.php` is not located in the DocumentRoot, it resides in `/var/www/templates`.

From the Apache config, we observe that requests to `/new/` are rewritten to `/var/www/templates/`. However, there is a catch: a `.html` suffix is automatically appended to any request within this directory. As a result, a request to:
* `https://hackcentral.darkfor.ge/new/getFlag.php`
will be rewritten to `/var/www/templates/getFlag.php.html`, which does not exist.

## Bypassing the .html Suffix with AcceptPathInfo

According to the [Apache documentation](https://httpd.apache.org/docs/2.4/mod/core.html#acceptpathinfo), the `AcceptPathInfo` directive determines whether Apache accepts trailing path information after a filename. When enabled (as is the default for script handlers like PHP), Apache will treat extra path segments as values for the `PATH_INFO` environment variable.

Since the challenge uses the official `php:8.2-apache` Docker image, which includes a PHP handler, we can exploit this behavior. By appending a trailing slash to our request, we can effectively bypass the `.html` suffix, as the rewritten path will now be interpreted as:
* `/var/www/templates/getFlag.php/.html`

This allows Apache to execute `getFlag.php` and pass `.html` via `PATH_INFO`.

To retrieve the flag:

1. Set the `session` cookie to the value found in `secret.txt`.
2. Make a request to:
   * `https://hackcentral.darkfor.ge/new/getFlag.php/`

![Flag!](/assets/images/62a8a67fb37370792c32c3e6dab4863da8b32768.png)

## Conclusion

This challenge demonstrates a combination of Apache misconfigurations, specifically DocumentRoot confusion and misuse of rewrite rules, paired with a subtle understanding of how Apache handles trailing path info. These bugs may seem innocuous individually, but when chained together, they enable full flag disclosure.

We’ll be releasing more CTF challenges soon, stay tuned!