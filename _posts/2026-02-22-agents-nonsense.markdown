---
layout: post
title:  "AGENTS.md doesn't make sense"
date:   2026-02-22 01:01:01 -0500
categories: AI
---

Trying to embrace the new era of AI-assisted engineering, and boy I'm worried! Mostly, about our inability to understand how the code really works, which is inevitable in the future where most of the code is produced by AI agents. A [counter-argument by Chris Lattner](https://www.modular.com/blog/the-claude-c-compiler-what-it-reveals-about-the-future-of-software) is that similar transition happened in the past when we stopped writing assembly. I think that worked because assembly translation is more mechanical and easier to test.

Anyway, the story starts with an "AGENTS.md" file that the author places in the repo. I don't think it makes any sense. Whatever you may want to put into it, there is a better place that is universal between AI and humans:
- description of what the project is about, and what it wants to be - should go into "README.md"
- code style, conventions, and organization - should go into "CONTRIBUTING.md"
- same goes for the testing procedures, dev environment, etc

What does make sense though, is a task or session specific permanent memory - "PLAN.md". It would contain my asks, important considerations, and the current progress along the goal. I start my work with it and keep it updated, not adding under version control.
