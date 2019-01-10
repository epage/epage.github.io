---
layout: remark.liquid
format: Raw
permalink: /{{parent}}/{{year}}/{{month}}/{{slug}}/
data:
  venue: Utah Rust
---
class: center, middle

# Deployment

---
class: center, middle

![XKCD: Python Environment](https://imgs.xkcd.com/comics/python_environment.png)

???


When selecting a blog engine, I did not want to touch anything that required an
installed interpreter.  That was one of my motivating reasons for getting
involved in cobalt.

---

# ./rg

---

# ./rg

- CI must run for all relevant platforms
- Can't edit in production ... but should you?

---
class: center, middle

# Performance

---

# xsv

???

How long does a batch operation take?

---

# xsv
# hg

???

How much lag is there between entering commands in my CLI UX take?

---

# xsv
# hg

- Development time
- Compile time

???

Scripting and experimental development are weaknesses

---
class: center, middle

# Powerful Ecosystem

???

Especially for being so young

---

# serde / structopt

---

# serde / structopt
# assert_cmd / assert_fs

---
class: center, middle

# Continual Improvement

---

# Path Gotchas

---

# Path Gotchas
# Rapid Prototyping

---

# Path Gotchas
# Rapid Prototyping
# Testing

---

# Path Gotchas
# Rapid Prototyping
# Testing
# Packaging

???

And of course hitting 1.0 on all of this
