---
layout: post
title: May 2024 Summary of Resources 
---

Summary of resources I read/watched in May. 
The focus of this month was to get familiar with terms and concepts underlying Generative AI. 

# 3Blue1Brown Videos
- [Attention in transformers, visually explained, Chapter 6, Deep Learning](https://www.youtube.com/watch?v=eMlx5fFNoYc)
- [But what is a GPT? Visual intro to transformers, Chapter 5, Deep Learning](https://www.youtube.com/watch?v=wjZofJX0v4M)

These two videos were helpful to gain an intuitive sense of how llms work. 
Lots of matrix multiplication though.
I also didn't quite understand what a full llm would look like end to end. 

# Brendan Bycroft's LLM Inference Visualization
- [Link](https://bbycroft.net/llm)

This visualization helped a lot. 
I feel like I got a better sense of the end to end picture of each of the stages of how llm inference works (at least for the simple architectures). 
The matrix operations are still hard for me to wrap my head around but that can probably only be solved with rewriting the architecture from scratch. 
There were also some basic probability concepts like mean and standard deviation that would probably be helpful to have some background knowledge on. 
Note that this only covers inference and pieces like the K/Q/V matrix weights and biases are already assumed to be pretrained.
This visualization also doesn't offer as much of an intuitive explanation for why pieces of the architecture exist (for example why layer normalization improves model stability).

# Andrej Karpathy's [1hr Talk] Intro to Large Language Models
- [Youtube video link](https://www.youtube.com/watch?v=zjkBMFhNj_g&t=1s)

This talk didn't explain any of the underlying details of how large languge models work but it provided a solid overview of the Generative AI space as a whole (with a strong emphasis on llms). 
There are several topics mentioned that I want to look into further:
- rlhf - optional third stage of training that uses human feedback to improve performance
- how multi modal models architectures differ from basic text to text models
- llm evals
- llm scaling laws - paper showing that performance increases predictably as the number of parameters of the size of the training text increases
- agentic reasoning