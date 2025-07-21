---
title: "Fixing Reverse Proxy for Plex with Nginx"
date: 2025-07-21
tags: ["nginx", "plex", "reverse-proxy"]
---

I started hosting my own media using plex, and I didn't wanted to have one url for each service, so I thought:
<!--more-->
_'Why don't I put everything under media.mydomain.com?'_

Then I started working on that, but Plex doesn't like my idea very much.
After a long time, I found a workaround.

```
    location /plex/ {
        rewrite /plex(/.*) $1 break;
        proxy_pass http://192.168.1.10:32400/web/;
        proxy_http_version 1.1;
        proxy_set_header Accept-Encoding "";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $http_host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        sub_filter '/web/' '/plex/web/';
        sub_filter_types html;
        sub_filter_once off;
    }
```

The **sub_filter** parameter saved my life. But only redirection /web to /plex/web wasn't enough, because soon after I found that solution, Plex pushed an update and broke it.

After some tweaks, I found that plex implemented a SRI (Subresource Integrity) on js and css files, so I added the second line, ```sub_filter_types html;```, to force my nginx to only filter html files, and leave css and js intact. That solved my issue.

You can find all my nginx.conf file on my github project: [nginx-media](https://github.com/celo0/nginx-media)
