---
layout: post
title: Lessons Learned from using HuggingFace for LLM Inference in Google Colab
---

My original goal was to set up an llm quickly in a google colab python notebook (because I didn't want to execute locally + I wanted access to an nvidia gpu quickly for free). 
I originally was looking at Ollama but its client/server architecture didn't seem to elegantly nicely with google colab. 
Side note, see this great [blog post](https://medium.com/@rifewang/analysis-of-ollama-architecture-and-conversation-processing-flow-for-ai-llm-tool-ead4b9f40975) for an understanding of Ollama's architecture.

I then turned to HuggingFace and quickly got into the weeds after running into several issues.
Here is a documentation of my experience.

## Notebook code
Here's the code:
```
pip install transformers
from transformers import pipeline
from huggingface_hub import login

token = "<set hf token here>"
login(token=token)

pipe = pipeline("text-generation", model="meta-llama/Llama-3.2-1B")

response = pipe("User: Hello! How are you?\nAssistant:", max_length=200)
print(response[0]["generated_text"])
```
The output generated is this (not great probably because its a 1B model):

```
Truncation was not explicitly activated but `max_length` is provided a specific value, please use `truncation=True` to explicitly truncate examples to max length. Defaulting to 'longest_first' truncation strategy. If you encode pairs of sequences (GLUE-style) with the tokenizer you can select this strategy more precisely by providing a specific strategy to `truncation`.
Setting `pad_token_id` to `eos_token_id`:None for open-end generation.
User: Hello! How are you?
Assistant: I am fine, thank you.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
User: I am also fine.
Assistant: I am also fine.
```

## Authentication
A lot of models on HuggingFace require accepting terms of service and because of that you need to provide some form of authentication.
What was easiest for me was creating an access token on the HuggingFace website and setting up the following code in my google colab notebook.
See [this doc](https://huggingface.co/docs/hub/en/security-tokens) for more details.

```
from huggingface_hub import login

login(token=<your login token>)
```

## Language Models vs Chat Models 
Hugging Face provides many kinds of models and because of this it's important to understand the various kinds of text transformer models (the category I was interested in) to fully understand the api.
Language models are models that are trained on lots of text using self supervised learning (where the objective can be determined from the inputs).
These language models are first pretrained on a large amount of text. 
They are then fine tuned for a specific task. 
Traditionally pretrained models aren't focused on a specific task because its harder to get fine tuned data in the large quantities needed for pretraining. 
Instead the knowledge from the pretrained model is transferred (i.e. using transfer learning). 

Causal language models continue text by outputting the next most likely tokens given the previous words. 
Chat models are a subtype of language model that continue chats.
The language model learns to chat (instead of being a text completer) by learning on fine tuned data.
The fine tuned data is in the format of high quality instruction and response pairs. 

This [Hugging Face doc page](https://huggingface.co/learn/nlp-course/chapter1/4#transformers-are-language-models) gives a good overview of the entire process.

## HuggingFace API
There's the `pipeline()` api which abstract away the manual chat formatting needed.
See an offical overview [here](https://huggingface.co/docs/transformers/en/conversations#what-happens-inside-the-pipeline).
```
# Use a pipeline as a high-level helper
from transformers import pipeline

pipe = pipeline("text-generation", model="meta-llama/Llama-3.2-1B")
```

Here's the code to generate a response:
```
response = pipe("User: Hello! How are you?\nAssistant:", max_length=200)
print(response[0]["generated_text"])
```

There exists [`AutoClasses` in the API](https://huggingface.co/docs/transformers/en/model_doc/auto#auto-classes) to load a model and tokenizer from a given path. 
Using these `AutoClasses`, we go one level deeper in the API. 

```
# Load model directly
from transformers import AutoTokenizer, AutoModelForCausalLM

tokenizer = AutoTokenizer.from_pretrained("meta-llama/Llama-3.2-1B")
model = AutoModelForCausalLM.from_pretrained("meta-llama/Llama-3.2-1B")
```

Note that previously when we were passing in message history we used a single string.
HuggingFace does provide a [chat templates](https://huggingface.co/docs/transformers/main/en/chat_templating) api that can be passed to the tokenizer class. 
Chat templates are part of tokenizer and convert list of messages into single tokenizable string.
Different models use different tokenizers.
Chat templates work using a jinja template ([source](https://huggingface.co/docs/transformers/main/en/chat_templating#advanced-how-do-chat-templates-work)). 
This template tells the tokenizer how to format the input conversation represented as a list of messages into a string that the model can understand. 

Note that not all models have this chat template feature.
It seems to be up to the owner of the model card to add an implementation. 
The way I check is by looking at the `tokenizer_config.json` file in the model card to see if there's a `chat_template` field. 
[Llama-3.2-1B](https://huggingface.co/meta-llama/Llama-3.2-1B/blob/main/tokenizer_config.json) does not appear to have one while [Mistral-7B-Instruct-v0.1](https://huggingface.co/mistralai/Mistral-7B-Instruct-v0.1/blob/main/tokenizer_config.json) does.

## Future exploration
- mixture of experts models although large, most parameters aren't active for each token generated so lower memory bandwidth. explore different types of model being adapted for different types of hardware
- ebpf for even deeper performance engineering analysis
- alternatives to hugging face. how does the business work. what does it solve. 
- fine tuning pretrained model: https://huggingface.co/learn/nlp-course/chapter3/1?fw=pt
    - this is where instructions are passed in
    - resource: https://huggingface.co/learn/nlp-course/chapter3/2
- Using RLHF after fine tuning pretrained model
- llama.cpp - how is this so fast locally

## Resources
- https://medium.com/data-science-at-microsoft/how-large-language-models-work-91c362f5b78f 