---
layout: post
title: Pairwise Comparisons
---
A foray into pairwise comparisons where a user is asked to ask a series of questions comparing two options at a time to determine the ranking of a list of options. 

## Problem
The context for this subject was that pairwise comparisons came up as a solution when discussing how to choose a baby's name. 
The idea is that human's are much better at choosing between two options which one is better than choosing from a large list.

## Initial Solution
The simplest solution is to generate all possible options, ask the user to compare each pair, and give a point to the winner. 

I created a quick draft of what this could look like in [this repo](https://github.com/andrew128/pairwise-comparisons).
There's a `main.py` file which takes inputs from the user to generate all possible options.
The core logic is in `pairwise_comparison_session.py` where the options and scores are stored. 
I store an indexing of an int to the string representation of the option and to the score.

A test class is added in `test_pairwise_comparison_session.py` for good measure.

## Discussion of Improvements

- Algorithmic scaling
    - Constructing a directed graph. Space will look like a bunch of (potentially separate) trees. 
    - Pruning: if you select option A as better than option B and option B as better than option C then you don't need to compare optoin A and option C (and potentially just trim option C completely)

- Technical Scaling
    - the maps of index to string and score can be stored inside of a sql database for processing many options

- Architectural improvements
    - perhaps use data classes instead of basic string
    - consider something else besides python. For example web site or app. Python was the easiest/fastest option.
    - provide the above optimizations as options that can be used to finetune to specific use cases