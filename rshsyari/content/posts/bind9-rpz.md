---
date: 2025-04-01T02:07:35+07:00
# description: ""
# image: ""
# lastmod: 2025-04-01
showTableOfContents: false
# tags: ["",]
title: "Bind9 Response Policy"
type: "post"
---



```zone
$TTL 3600
@	SOA	localhost. admin.localhost. ( 3 3600 600 3600 600 )
@	NS	localhost.

@	A	127.0.0.1
*	A	127.0.0.1
```


```
response policy {
	zone "nothere.com" policy CNAME www.example.com;
};
```
