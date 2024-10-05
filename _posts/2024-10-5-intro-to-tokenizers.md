---
layout: post
title: Intro to Tokenizers
---

Notes on the tokenizer space, what they are, what problems they solve, some example code, and some areas to explore in the future.

## What is a tokenizer?
A tokenizer is a way of turning input text into model input. 
Note that the way you tokenize the input text should be consistent with the pretraining tokenization of the model. 
Otherwise, the model will not "understand" the input text the way it was trained.
There are various ways of splitting up the input text into smaller pieces that are called words or subwords. 
These are then mapped to an id through a look up table (e.g. index).

All tokenizers implement the `encode` and `decode` method.
- `encode` takes in a string and returns a list of tokens
- `decode` takes in a list of tokens and returns a string

A good tokenizer will:
- have a small, consistent vocabulary: this is extremely important as larger vocabularies increase the size of the model
- represent words with similar meaning close to each other in the embedding space

## Show me the code

### Karpathy's nanoGPT
In [Andrej's example](https://www.youtube.com/watch?v=kCc8FmEb1nY), he made a simple character level tokenizer.
Code copied from [his repo](https://github.com/karpathy/ng-video-lecture/blob/52201428ed7b46804849dea0b3ccf0de9df1a5c3/gpt.py#L25C1-L32C101).

```
# here are all the unique characters that occur in this text
chars = sorted(list(set(text)))
vocab_size = len(chars)

# create a mapping from characters to integers
stoi = { ch:i for i,ch in enumerate(chars) }
itos = { i:ch for i,ch in enumerate(chars) }
encode = lambda s: [stoi[c] for c in s] # encoder: take a string, output a list of integers
decode = lambda l: ''.join([itos[i] for i in l]) # decoder: take a list of integers, output a string
```

This code simply constructs a mapping of characters to integers and vice versa for the encode and decode functions.

This is not ideal because its character based and it is not able to encode meaning or context between similar words.

### BPE Tiktoken
This algorithm is based off of chunks which are able to encode meaning and context between similar words. 
It's in between character (covered in previous section) and word level tokenization.
Word level tokenization can result in massive vocabularies while also not encoding meaning or context between similar words.

[HuggingFace's tutorial](https://huggingface.co/docs/transformers/en/tokenizer_summary) does a good job of explaining how BPE works. 
BPE starts out by counting the frequency of tokens associated with specific words. 
At a high level it continually splits and combines the most frequent pair of tokens in the corpus until the desired vocabulary size is reached.
In the end each token in the corpus (now representing the vocabulary) is associated with a unique id.

## Further areas to explore
- Andrej Karpathy's has a video [Let's build the GPT Tokenizer](https://www.youtube.com/watch?v=zduSFxRajkE&t=85s) that expands farther into the topic.

- There are also many types of tokenization strategies (e.g. WordPiece, SentencePiece). 

- Tokenizers are only meant for text but there are also other modalities that also require tokenization, such as images, multimodal, audio, and video. 
See [this tutorial](https://huggingface.co/docs/transformers/en/preprocessing) for starter information. 

## Resources
- https://huggingface.co/docs/transformers/en/tokenizer_summary 
- https://huggingface.co/docs/transformers/en/preprocessing
- https://www.youtube.com/watch?v=zduSFxRajkE&t=85s 
- https://github.com/openai/tiktoken 