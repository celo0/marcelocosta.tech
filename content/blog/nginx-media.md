---
title: "Self-Hosting Plex Under One Domain with Nginx (The Real Fix)"
date: 2025-07-21
tags: ["nginx", "plex", "reverse proxy", "self-hosting", "home lab", "media server", "sub_filter", "sri", "nginx config"]
---

I started self-hosting my media using Plex, and I didn’t want a different URL for every service. So I thought:
<!--more-->
> _"Why not just put everything under `media.mydomain.com`?"_

Simple idea. Plex, however, didn’t agree.

I began setting up Nginx to reverse proxy everything under subpaths like `/plex/`, but Plex was stubborn. After a lot of trial and error, I finally found a workaround:

```nginx
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
The `sub_filter` directive saved my setup. Plex hardcodes `/web/` all over the place, and I needed to rewrite it to `/plex/web/`. That worked — until Plex pushed an update.

Turns out, Plex uses [Subresource Integrity (SRI)](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity), and my filters were breaking the integrity of JS and CSS files. The fix? I added this line:

```nginx
sub_filter_types html;
```

This tells Nginx to only apply the substitution to HTML files, leaving the JS and CSS untouched. That did the trick.

If you want to check out the full `nginx.conf`, it’s on my GitHub:  
👉 [nginx-media](https://github.com/celo0/nginx-media)