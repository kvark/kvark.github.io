---
layout: post
title:  "Feeding the data monster"
date:   2026-07-12 02:02:02 -0500
categories: ai robotics
---

Prompted by presentations from [Humanity & AGI Summit 2026: Robotics for Future Civilization](https://luma.com/wkolq6uc?tk=aUgmgA), I sketched out a few strategies to train the robots. I don't do this at work and by no means express the opinions of my employer here. The distinct strategies on my list are presented in the order from more realistic to more distant fantasy.

The lack of data is often named as one of the showstoppers for robotic intelligence. Solving it the same way as Tesla solved FSD doesn't appear possible: there is little use of fully human-controlled robots outside of some very specific areas (like surgery). I'm a big fan of simulation, but I'm going to put it aside for the purpose of this investigation - it deserves its own post and I don't want to share my day-job work here. Also skipping tele-op since it scales poorly and targets a specific robot only.

# Public internet videos

This is the current approach for many AI startups, at least a starting point for Skild AI and Rhoda AI. It's easy to find a lot of data but hard to extract value. The goal is to train a world model that understands physics and causality. It has to be smart enough to then translate to a bot, and that translation is the hard part of the approach. Once the big world model exists, we can distill it to a small VLA to run on the bot directly as the policy, or manage to run the model directly on the bot.

Notably, it's a giant model that is trained in an explicitly constructed loop: engineers gather more data, clean it up, throw it into the oven, get the updated model out, etc. This isn't AGI by any means, since the training loop is heavily assisted by humans.

# Augmented vision

See [my AR explosion post]({% post_url 2026-07-12-ar-explosion %})

Once people start wearing glasses, an interesting opportunity arises for robotics. Imagine being paid for sharing your (anonymized) recording together with your voice annotations of what you are doing. Practically golden labelled data right there, very easy to collect. Once these glasses are popular, and the data collection pipeline exists, we can start training models directly on it with behavioral cloning. Of course, there is still a domain gap to cross between the glasses' cameras with human hands to the concrete robotic eyes and hands, but it's a much easier bridge to cross than the wild internet data.

# Self-training

Now we are getting more into sci-fi territory. If the robot is failing to assemble the Rubik's cube, maybe the best solution is not to gather more data, post-train, and re-export and re-deploy the model. This is way too slow of a development cycle, especially if we want a robot to be useful in novel situations. Colonizing Mars anyone?

What if the robot instead could learn on the spot? As long as it can compute the rewards, and it has the necessary hardware to do backpropagation (assuming the network architectures and hardware are roughly the same as today), it can do real-life reinforcement learning. A robot would accumulate knowledge and skills as it operates, occasionally sharing its best findings with others or a central management entity.

Admittedly, rewards are the hard part here, and I don't have the magic bullet for it. I suspect the answer lies in following the early development of humans. Intrinsic motivation through curiosity, novelty, play, plus explicit feedback from a "parent".

Live learning is something that I find extremely important for agentic physical AI. Not many companies even attempt to solve this problem. Everybody is focused on giant offline models and inference-only devices. If that direction kicks off, I'm curious how it's going to drive the hardware development for AI inference and training: GPUs are not very efficient at this today, they spend most of their energy moving data around.
