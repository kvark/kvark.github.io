---
layout: post
title:  "Inference Arena"
date:   2026-04-04 01:01:01 -0500
categories: ai performance
---

Looking at the screen, seeing my free space dropping by 20 GB from entering a Python development NixOS shell. It carries a few hand-selected ML libraries: pytorch, torchvision, transformers, onnx, and some more. That's a ton of code! I wonder how it fares against smaller and newer libraries, including those made with Rust language.

# Infenera

Here comes [Infenera](https://inferena.tech/) (short for "Inference Arena") - my benchmark of ML framework inference and training. We are considering 5 standard models to start with:

- SmolLM2 as a language model (SLM, not LLM?)
- SmolVLA as a vision-language model
- StableDiffusion as a generative model
- Resnet50 as a classification model
- Wisper-tiny as a speech recognition model

The idea is to cover the ground for different types of ML components used in these models: transformers with different kinds of attention, diffusers, convolution layers, MLP, and so on. For each model, we are interested in the following aspects:

1. time per forward propagation - measuring *inference* throughput
2. end-to-end time - measuring *latency*
3. time per backward propagation - measuring *training* throughput

We've got a collection of ML frameworks to test on:

- starting with portable giants like PyTorch, JAX, and ONNX RT
- adding prominent small players like [GGML](https://github.com/ggml-org/ggml) with its *llama.cpp* + *whisper.cpp*
- with modern Rust-based frameworks like [Burn](https://github.com/tracel-ai/burn), [Candle](https://github.com/huggingface/candle)
- and even underground [Luminal](https://github.com/luminal-ai/luminal) and [Meganeura](https://github.com/kvark/meganeura)
- concluding with platform-specific frameworks like Apple's MLX

To instill trust in the results, we check for the loss to match PyTorch in both forward and backward passes. Results are crossed out if the loss is above 10% - it's considered that a candidate ML framework worked with a different model in these cases.

# Results

Firstly, PyTorch is very solid in performance. It's good all around - pretty sure nobody is going to be fired by choosing PyTorch any time soon.

Secondly, the difference in performance can be astonishing. When working on games, I tend to think about relative milliseconds. When working with compression algorithms back in the day, even 0.1% was a significant improvement. Here in ML land, if your data misses the chance to be properly on-chip when accessed, you immediately see 2x, 5x, and sometimes 10x difference. The performance curve is extremely sharp and sensitive to optimization.

Thirdly, Apple is doing pretty well with their MLX framework, beating PyTorch on home ground. They may be struggling with actual models, but the infrastructure to run and train them is solid.

Lastly, ML is not as accessible as I assumed. For instance, the Framework 12 that I got some 6 months ago doesn't get an accelerated XPU device in PyTorch. Another mini-PC I have with Radeon 780M doesn't support ROCm. And we haven't even tried any of the mobile devices yet!
