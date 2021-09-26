---
layout: post
title:  "Taming software complexity"
date:   2021-09-26 11:11:11 -0500
categories: engineering
---

The software engineer's job is about taming the complexity.

There is an opinion that complexity never goes away. Engineers can just split it and solve in pieces, gluing them together. An example given by Don Norman in [Living with Complexity](https://www.youtube.com/watch?app=desktop&v=flRuSn0df8Q), the magic of making good coffee can be contained at the level of individual capsules. This made a lot of sense to me, and I concluded that "divide and conquer" is the general approach to problem solving.

Recently I realized that this model doesn't hold. Complexity doesn't behave like a body of water that keeps the same volume no matter how you split it. Instead, software complexity needs to be seen in the context of Kolmogorov complexity:
> In algorithmic information theory, the Kolmogorov complexity of an object, such as a piece of text, is the length of a shortest computer program that produces the object as output.

More concretely, I propose a mental model to think of software complexity as a number. And solving is would mean finding the simplest way to compute or represent it. For example, number `59049` can be represented as `59 * 1000 + 7^2`. Or it can just be `3^10`. There is no "zero sum" here, there can be just really nice solutions that aren't obvious. We are not dividing, we are finding a good representation. In other words, we are "discovering" complexity. Our point of view may determine what the complexity is.

How do we apply this mental model to practice? There is no general solution to finding the Kolmogorov complexity! I've spent wonderful years [digging into](https://encode.su/threads/652-DARK-a-new-BWT-based-command-line-archiver) lossless compression based on Burrows-Wheeler Transformation. One of the lessons learned is that good compression is based on insights about the data. Compression is about expressing the knowledge about the data in the code, without over-fitting.

Same applies to software engineering: we need to "understand" (as in [A Theory of Mind Based on Data Compression](http://ceur-ws.org/Vol-1419/paper0045.pdf)) the task deeply. Finding the internal structure, connections between different parts of the system, opens new ways of representing the complexity. Over-fitting would mean being unable to change the scope of the problem later on, building an architecture that is too rigid, which will eventually need to be scraped.

The bottom line of this essay is that the main task of engineering - managing complexity - is essentially a form of art. There is no methodology that would allow to reach the proximity of an optimal solution (like "divide and conquer"). Best engineering solutions are truly marvels of human thinking.
