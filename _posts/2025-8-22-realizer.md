---
layout: post
title: Generating Minecraft Structures With Natural Language
---

![_config.yml]({{ site.baseurl }}/images/castle.png)

Example output from query "gothic castle". 

## What exactly is this?
I thought about this idea a few months ago after trying to play Minecraft and discovering that I wanted to explore complex worlds and generate complicated structures while at the same time being unwilling to dedicate the hours to manually construct those worlds (even with world edit mods). 
The natural follow up was to think about using generative ai techniques to quickly construct worlds from one's imagination. 

This mod is my initial exploration into that area.
Specifically, this mod creates structures based off of natural language input. 

## Text to Voxel Model Approach?
When I initially thought to create this mod, I thought that I would have to create a text to voxel model approach where I would be collecting lots of training data (an unappealing thought) and training a model (probably starting off with a simple MLP and eventually iterating to a transformer based model). 
There are at least a couple of papers out there (e.g. [Voyager](https://voyager.minedojo.org/)) that may be used to generate "synthetic" data where I prompt them to execute structures. 
This would potentially make the modeling approach more tractable. 

Just as I was about to pursue this route, I thought about how easily Claude created my python script that generated a simple 3x3x3 stone block structure.
Wondering how far Claude could go, I asked Claude many queries (e.g. a gothic castle, a golden rubber duck) and after seeing the high quality results, decided to go this route.
My guess as to why Claude is so good at this is that Claude is trained on all of github/gitlab and has therefore seen all the open source minecraft code and mods. 

## A side note on the Minecraft Mod Ecosystem
Despite hearing about and trying out mods (mostly on a mac) many years ago, this was my first time writing a mod.
I initially tried using Forge since it seemed more mature but ran into several mac related build issues and decided to try out Fabric which was much smoother. 
Given how long Minecraft mods have been around I thought that certain basic actions like placing a pregenerated structure would've been very well documented. 
Not sure if this is a Fabric specific problem but without GPT telling me to use the StructureTemplateManager class (and me subsequently looking at the deobfuscated Minecraft source code and public github repos) and it would've been very difficult for me to piece together how to place and generate structures in nbt format. 

One thing claude could improve on is going into references like java .class files and understanding the code.Tthis was a major blocker in that it probably just assumed how StructureTemplateManager worked. I was able to eventually fix all of these issues by copy pasting the entire file into Claude's context. 

## Future work
- UX improvements: loading bar, undo previous edit command
- select chunks of the world (look at how Axiom and Worldedit mods implement selection) and edit those
- generating worlds (not just structures) with natural language
- As I'm writing this Genie3 was just released. This coupled with the [GameNGen Doom  project](https://gamengen.github.io/) shows a very interesting area of exploration that I'm interested in exploring further: the idea of generating whole new worlds that one can generate with a single prompt without spending hundreds of hours learning and building with complex tools like Blender. 