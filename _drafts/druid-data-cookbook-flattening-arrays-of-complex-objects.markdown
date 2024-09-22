---
layout: post
title:  "Druid Data Cookbook: Flattening Arrays of Complex Objects"
categories: blog apache druid imply sql tutorial
twitter:
  image: /assets/2021-12-21-elf.jpg
---

![Druid Cookbook](/assets/2021-12-21-elf.jpg)

A common problem in data that we ingest into Druid is that we may encounter arrays of nested objects, and we want to reason about specific fields within those objects. For instance, assume we have various teams for some sort of contest, and the members of a team might be represented like so:

```json
[
  {
    "name": "Alice",
    "gender": "F"
  },
  {
    "name": "Bob",
    "gender": "M"
  },
  {
    "name": "Carol",
    "gender": "F"
  }
]
```

Each member has a name and some other attributes. But what if I want to get a list of all the teams that Bob is a member of? I'd need to extract an array of the relevant subfields only. How do I do this in Druid?

The naïve approach would be to write some expression like `JSON_VALUE("members", '$[*].name')`. But unfortunately, Druid does not support wildcard syntax in JSONPath expressions. So if seems that we are stuck. Or are we?

