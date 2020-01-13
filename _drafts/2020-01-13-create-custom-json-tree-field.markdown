---
layout: post
title:  "Create custom tree field for json data"
date:   2020-01-13 00:12:33 +0700
tags: [odoo,custom field]
---

Once in a while, instead of the usual one to many relationship, it can be easier to stored data as a lump of json. 

Trade off: 
Lower concurrency. Due to lock on the whole record.