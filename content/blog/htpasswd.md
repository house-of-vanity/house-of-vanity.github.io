+++
title = ".htpasswd one-liner"
date = "2020-07-13"
description = "creating password hash for Basic auth"

[taxonomies]
tags = ["linux", "tools", "selfhosting"]

[extra]
author = { name = "@ultradesu", social= "https://github.com/house-of-vanity" }
+++

It's annoying when you need apache2-utils just for creating password hash for Basic auth. So here is Shell one-liner doing it using openssl.
```sh
user=ab
pass=pwd
printf "${user}:$(openssl passwd -apr1 ${pass})\n"
```
---
