---
layout: post
title: LLM Memory and Latency Profiling
---

This post covers profiling an LLM, something I've wanted to do for a while.

## Google Colab Machine Specs
I'm choosing to use Google Colab because it gives access to an NVidia T4 GPU for free quickly. 
[Here's a gist](https://gist.github.com/andrew128/bcc49798e2b1f6f6ec3a3a048f26fe59) where I print information using python libraries to learn more about the environment.

[This](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/6/html/deployment_guide/s2-proc-cpuinfo#s2-proc-cpuinfo) site gives an overview of `/proc/cpuinfo` where the data is being read from. 
The way the contents are generated varies by architecture. 
[Here](https://github.com/torvalds/linux/blob/master/arch/arm64/kernel/cpuinfo.c?utm_source=chatgpt.com) is a code pointer for how Arm64 on Arch Linux reads info. 
[Here](https://elixir.bootlin.com/linux/v5.14.13/source/arch/x86/kernel/cpu/common.c#L641) is how x86 on Arch Linux reads the cpu info. 
This information is fetched by the Linux kernel using Microcode (layer within CPU that interprets machine instructions and orchestrates hardware to execute them) using a CPUID instruction (special instruction to query processor for details) and 
The kernel executes the CPUID on startup and stores the result in structure like [cpuinfo_x86](https://elixir.bootlin.com/linux/v5.14.13/source/arch/x86/include/asm/processor.h#L81). 
The kernel also reads Model-Specific Registers ([MSR's](https://en.wikipedia.org/wiki/Model-specific_register)) which are special registers that contain more information. 

This notebook has a single Xeon CPU processor with a clock speed of 2 GHz meaning it can run 2 billion CPU cycles per second.
It has one physical core but 2 siblings meaning that the CPU can use hyperthreading (i.e. 1 physical core can act as 2 logical cores by sharing resources for two threads simultaneously).
It has a cache size of 56 mb. 

Using the nvidia smi library, we can see that we have a T4 connected with ~15GB of vram. 

## 1B Model Profiling

I picked `Llama 3.2 1b`, a smallish chat model to run quickly with the HuggingFace Transformer's API.
[Here](https://andrew128.github.io/HuggingFace/) is a blog post I wrote exploring the HuggingFace API.

```
pipe = pipeline("text-generation", model="meta-llama/Llama-3.2-1B")
response = pipe("User: Hello! How are you?\nAssistant:", max_length=200)
```

At a high level, let's go over what is happening in the code above.
Weights are downloaded to disk and loaded into main memory by
deserializing model checkpoint into usable tensors and then
transferred to GPU memory.

Input text is tokenized into tensors in RAM and transferred to GPU.
Model weights and input tensors are used in matrix multiplications
and other operations on GPU. Intermediate computations like activations
are stored in GPU memory. After inference results are transferred back
to RAM.

## Time Profiling with cProfile
A snippet of my code measuring how long the pipeline took. 
It measured 13 seconds. 
Let's see if we can dig deeper into exactly what is taking so long. 

```
start_time = time.time()

pipe = pipeline("text-generation", model="meta-llama/Llama-3.2-1B")
response = pipe("User: Hello! How are you?\nAssistant:", max_length=200)
print(response[0]["generated_text"])

end_time = time.time()

print(f"Execution time: {end_time - start_time} seconds")

```

Let's now use [cProfile](https://docs.python.org/3/library/profile.html), which tells us how long specific functions are executed for and how many times they are called. 

```
def profile_code():
    pipe = pipeline("text-generation", model="meta-llama/Llama-3.2-1B")
    response = pipe("User: Hello! How are you?\nAssistant:", max_length=200)
    return response[0]["generated_text"]

profiler = cProfile.Profile()
profiler.enable()
profile_code()
profiler.disable()

profiler.dump_stats('profile_data.prof')

# Memory usage
# mem_usage = memory_usage((profile_code,), interval=0.1)
# print(f"Memory usage over time: {mem_usage} MB")
# print(f"Peak memory usage: {max(mem_usage)} MB")

# Time Usage
stats = pstats.Stats('profile_data.prof')
stats.sort_stats('calls', 'time').print_stats(20) 
stats.sort_stats('time', 'calls').print_stats(20) 
```

Here is the output:
```
Wed Dec 25 05:34:06 2024    profile_data.prof

         794045 function calls (692599 primitive calls) in 11.434 seconds

   Ordered by: call count, internal time
   List reduced from 1171 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    97621    0.029    0.000    0.029    0.000 {method 'startswith' of 'str' objects}
    91037    0.103    0.000    0.103    0.000 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:1918(__getattr__)
    43642    0.052    0.000    0.052    0.000 {built-in method torch._C._get_tracing_state}
40446/189    0.160    0.000    3.933    0.021 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:1740(_call_impl)
40446/189    0.078    0.000    3.934    0.021 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:1732(_wrapped_call_impl)
35819/32473    0.010    0.000    0.018    0.000 {built-in method builtins.isinstance}
    21357    0.727    0.000    0.727    0.000 {built-in method torch._C._nn.linear}
    21357    0.062    0.000    0.816    0.000 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/linear.py:124(forward)
    17744    0.004    0.000    0.004    0.000 {method 'get' of 'dict' objects}
13486/13286    0.007    0.000    0.028    0.000 {built-in method builtins.len}
    13415    1.159    0.000    1.159    0.000 {method 'to' of 'torch._C.TensorBase' objects}
    12631    0.266    0.000    0.266    0.000 {built-in method torch.cat}
    12285    0.044    0.000    0.044    0.000 {method 'transpose' of 'torch._C.TensorBase' objects}
    12096    0.049    0.000    0.049    0.000 {method 'view' of 'torch._C.TensorBase' objects}
    11561    0.002    0.000    0.002    0.000 {method 'items' of 'dict' objects}
9003/1410    0.012    0.000    0.012    0.000 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:2778(named_modules)
8474/8472    0.002    0.000    0.003    0.000 {built-in method builtins.getattr}
     8032    0.001    0.000    0.001    0.000 /usr/lib/python3.10/inspect.py:2687(name)
     8008    0.001    0.000    0.001    0.000 {method 'values' of 'collections.OrderedDict' objects}
     7949    0.009    0.000    0.017    0.000 {built-in method builtins.hasattr}


Wed Dec 25 05:34:06 2024    profile_data.prof

         794045 function calls (692599 primitive calls) in 11.434 seconds

   Ordered by: internal time, call count
   List reduced from 1171 to 20 due to restriction <20>

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
      146    3.174    0.022    3.174    0.022 {method 'copy_' of 'torch._C.TensorBase' objects}
    13415    1.159    0.000    1.159    0.000 {method 'to' of 'torch._C.TensorBase' objects}
        1    0.826    0.826    0.826    0.826 {built-in method from_file}
      190    0.729    0.004    0.731    0.004 /usr/local/lib/python3.10/dist-packages/transformers/generation/utils.py:2423(_has_unfinished_sequences)
    21357    0.727    0.000    0.727    0.000 {built-in method torch._C._nn.linear}
        2    0.474    0.237    0.474    0.237 {method 'read' of '_ssl._SSLSocket' objects}
     6237    0.363    0.000    0.724    0.000 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:70(forward)
     3024    0.281    0.000    0.654    0.000 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:203(apply_rotary_pos_emb)
    12631    0.266    0.000    0.266    0.000 {built-in method torch.cat}
     6048    0.209    0.000    0.354    0.000 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:196(rotate_half)
    231/1    0.196    0.001    1.325    1.325 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:897(_apply)
     3024    0.172    0.000    2.084    0.001 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:491(forward)
     3024    0.169    0.000    3.665    0.001 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:601(forward)
40446/189    0.160    0.000    3.933    0.021 /usr/local/lib/python3.10/dist-packages/torch/nn/modules/module.py:1740(_call_impl)
     6049    0.158    0.000    0.158    0.000 {method 'reshape' of 'torch._C.TensorBase' objects}
     3024    0.127    0.000    0.127    0.000 {built-in method torch._C._nn.scaled_dot_product_attention}
     6237    0.126    0.000    0.126    0.000 {method 'mean' of 'torch._C.TensorBase' objects}
     6048    0.122    0.000    0.314    0.000 /usr/local/lib/python3.10/dist-packages/transformers/models/llama/modeling_llama.py:246(repeat_kv)
     6237    0.120    0.000    0.120    0.000 {method 'pow' of 'torch._C.TensorBase' objects}
        1    0.117    0.117    3.757    3.757 /usr/local/lib/python3.10/dist-packages/transformers/models/auto/auto_factory.py:447(from_pretrained)


<pstats.Stats at 0x7bb60f873370>
```

Looking at the data there's a single `from_file` function call that takes a long time. 
This is most likely reading the downloaded weights from RAM to the GPU's memory. 
There are also two `read` operations on SSL Sockets. 
These are most likely related to downloading the model weights from the HuggingFace repository. 

Also there's the torch copy and to functions from TensorBase. 
Unfortunately it's not clear from this what if anything can be done to reduce the number of operations, especially from such a high level api. 

## Analyzing Main Memory Usage

We can use this code to profile main memory usage:
```
# Query every 1 second
mem_usage = memory_usage((profile_code,), interval=1)
print(f"Memory usage over time: {mem_usage} MB")
print(f"Peak memory usage: {max(mem_usage)} MB")
```

Here's the output:
```
Memory usage over time: [2913.14453125, 2913.14453125, 4477.9921875, 5004.890625, 7706.671875, 6478.078125, 6512.20703125, 4166.12109375, 3398.07421875, 3398.07421875, 3398.07421875, 3398.07421875, 3398.07421875, 2913.1953125] MB

Peak memory usage: 7706.671875 MB
```

Not super helpful.
Ideally we can see what the operations are.
It's not clear what operations are using how much memory.
In addition, this library only measure RAM and the core operations are performed on the GPU. 

## Analyzing GPU Usage
Here's the PyTorch profiler library code:
```
from torch.profiler import profile, ProfilerActivity, record_function

with profile(activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA]) as prof:
    profile_code()

print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))
```

Unfortunately I'm using the free version of Google Colab and was not able to use terminal to use Tensorboard. 
Here's the text output:
```
Before pipeline creation - GPU memory allocated: 8.12 MB
Device set to use cuda:0
Truncation was not explicitly activated but `max_length` is provided a specific value, please use `truncation=True` to explicitly truncate examples to max length. Defaulting to 'longest_first' truncation strategy. If you encode pairs of sequences (GLUE-style) with the tokenizer you can select this strategy more precisely by providing a specific strategy to `truncation`.
Setting `pad_token_id` to `eos_token_id`:128001 for open-end generation.
Before inference - GPU memory allocated: 4722.39 MB
After inference - GPU memory allocated: 4722.39 MB
-------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                                                   Name    Self CPU %      Self CPU   CPU total %     CPU total  CPU time avg     Self CUDA   Self CUDA %    CUDA total  CUDA time avg    # of Calls  
-------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
                                           aten::matmul         1.50%     220.802ms        10.67%        1.575s      73.119us       0.000us         0.00%        3.847s     178.532us         21546  
                                           aten::linear         0.89%     131.999ms        13.02%        1.921s      89.962us       0.000us         0.00%        3.846s     180.092us         21357  
                                               aten::mm         5.75%     848.979ms         7.94%        1.172s      54.869us        3.846s        68.81%        3.846s     180.092us         21357  
std::enable_if<!(false), void>::type internal::gemvx...         0.00%       0.000us         0.00%       0.000us       0.000us        3.809s        68.14%        3.809s     179.288us         21245  
                                            aten::copy_        40.93%        6.041s        50.54%        7.459s     858.901us        1.279s        22.88%        1.279s     147.296us          8684  
                                               aten::to         0.11%      16.601ms         9.00%        1.328s      60.602us       0.000us         0.00%        1.242s      56.645us         21919  
                                         aten::_to_copy         0.04%       5.468ms         8.89%        1.312s       2.201ms       0.000us         0.00%        1.242s       2.083ms           596  
                       Memcpy HtoD (Pageable -> Device)         0.00%       0.000us         0.00%       0.000us       0.000us        1.241s        22.20%        1.241s       7.432ms           167  
                     aten::scaled_dot_product_attention         0.41%      60.214ms         2.29%     338.007ms     111.775us       0.000us         0.00%     111.852ms      36.988us          3024  
          aten::_scaled_dot_product_efficient_attention         0.29%      43.443ms         1.88%     277.793ms      91.863us       0.000us         0.00%     111.852ms      36.988us          3024  
-------------------------------------------------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  ------------  
Self CPU time total: 14.759s
Self CUDA time total: 5.590s
```

## GPUs, Flame Graphs, and eBPFs
Above it's hard to determine where in the program a long call is happening.
There is no unified place to delve deeper into the call stack. 

I remember coming across eBPFs a while ago as a way of digging really deep into the stack for CPU profiling and Flame Graphs as a way of visualizing the data.
I was excited to come across [this AI Flame Graph blog post](https://www.brendangregg.com/blog/2024-10-29/ai-flame-graphs.html) but unfortunately it isn't publicly available at the moment. 

## Optimizations
In terms of optimizations, optimizing throughput for many requests will be more interesting.
Some ideas:
- caching responses for handling many similar requests
- async processing for handling longer term requests
- concurrently handling multiple requests
- batch inference

As for next steps, I'll be looking into implementing an llm "from scratch" using PyTorch and then try to actually measure and compare optimizations. 
This will allow me to learn mroe about the lower level concepts.

## Further areas to explore
- CPU Design
    - CPU core hyperthreading
    - specter and meltdown
    - Just in time (LLVM, compilers)
    - Apple M3 chip
- Interaction between hardware and software
    - [Microcodes](https://en.wikipedia.org/wiki/Microcode)
    - [Reprogramming CPU microcode with an Arduino](https://www.youtube.com/watch?v=JUVt_KYAp-I&ab_channel=BenEater)
- Inference hardware designs
    - Groq
    - Cerebras
- CPU Flamegraphs