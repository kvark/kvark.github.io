---
layout: post
title:  "Rusty Object Notation"
date:   2017-08-09 11:11:11 -0500
categories: format data json
---

JavaScript. The practice-oriented language made scripting the Web possible for millions of programmers. It grew an ecosystem of libraries, even started attacking the domains seemingly independent of the Web, such as: native applications (Node.js) and interchange formats (JSON).

There is a lot not to like in JSON, but the main issue here is the lack of semantics. JavaScript doesn't differentiate between a map and a struct, so any other language using JSON has to suffer. If only we had an interchange format made for a semantically strong language, preferably modern and efficient... like Rust. Here comes Rusty Object Notation - [RON](https://github.com/ron-rs/ron).

RON aims to be a superior alternative to JSON/YAML/TOML/etc, while having consistent format and simple rules. RON is a pleasure to read and write, especially if you have 5+ years of Rust experience. It has support for structures, enums, tuples, homogeneous maps and lists, comments, and even trailing commas!

We are happy to announce the release of RON library [version 0.1](https://crates.io/crates/ron/0.1.1). The implementation uses `serde` for convenient (de-)serialization of your precious data. It has already [been accepted](https://github.com/amethyst/amethyst/pull/269) as the configuration format for Amethyst engine. And we are just getting started ;)

RON has been designed a few years ago, to be used for [a game](https://github.com/kvark/claymore) no longer in development. The idea rested peacefully until one shiny day [torkleyy](https://github.com/torkleyy) noticed the project and brought it to life. Now the library is perfectly usable and solves the question of readable data format for all of my future projects, and I hope - yours too!
