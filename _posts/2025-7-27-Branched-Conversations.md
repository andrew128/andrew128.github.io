---
layout: post
title: Branched Conversations
---

tldr engage with LLMs in branched conversations

## Inspiration
This is an exploration of an idea I've had for a while now. 
Most recently it came up when I was learning about DCFs.
At the surface DCFs seem simple but I had many follow up questions for each piece of the model, trying to figure out what the assumptions being made were as well as how each made intuitive sense. 
I mostly used ChatGPT's web interface but found it limiting in that I wanted to ask many follow up questions to a particular AI response and could not easily track everything.
Instead I would have to open many tabs for follow up questions but it was too annoying to try and copy paste or manually summarize the previous chat context. 

The most natural solution was to create a branched UI as seen here:

![_config.yml]({{ site.baseurl }}/images/branched_conversations.png)

There's definitely a lot more that can be pursued here and I think this demonstrates one example of how different chat interfaces can differ. 

## Architecture
I made an app that can converse with various providers (e.g. Ollama, OpenAI) from a simple chat interface.
A local sqlite database is used for storing everything from messages to provider configs. 
I wanted a local first app that could operate offline (i.e. with Ollama) while still being able to make the more complex web UIs. 
Electron was therefore the natural choice. 
This is also what the Claude Mac App uses. 

## Branched Conversation
The core problem unique to this project was the branched conversation idea.
More specifically, there were design decisions I made about how to store and display this data. 

### Storage
There were several options I considered when thinking about how to store branched conversation data. 
One was to use sqlite's recursive CTEs to self reference messages. 
Another was to store adjacency lists as a string for a given message. 
I ultimately went with adjacency lists after thinking more about the type of workload this app would encounter:
- write heavy but only to leaf nodes of the single existing tree for a given chat
- read heavy, usually from the leaf straight to the root
- this data needs to be persisted

With this method, sqlite has a messages table with a "path" column.
The path, which stores the path from the parent to the root, is represented by a string with all of the message ids from the parent to the root separated by a "/". 
In this way reads automatically know the path from a given leaf message to the root just by fetching the leaf message's row. 
Writes are simple - we just concatenate to the parent's path field when a new message is generated. 
This also fits nicely into the local sqlite db format (as opposed to using something more complicated like Neo4j).  

### Display
The flip side is displaying the branched messages. 
It turns out that there are many such tree balancing algorithms. 
I went with the Reingold Tilford Algorithm. 
As of writing no wikipedia page exists for this algorithm but I was able to understand and have Claude implement this for me relatively easily. 
The general idea is to calculate the x and y coordinates for each node in a given tree to display the tree in an aesthetically pleasing way. 

## Experience with Claude Code
This was my first major experience with Claude Code. 
While Claude Code is definitely far from one shotting an entire application in one go (even one that I would say is relatively simple like this), it definitely saved me on the order of weeks throughout this project. 
Because it was able to structure/scaffold most of the code, I just had to steer the model in the right direction which allowed me to save my brain power for more interesting problems. 
The pattern I eventually found myself repeating that worked well was to have prompt with an extremely detailed feature specific (with an image from Excalidraw for UIs). 
It would create a plan that I would review and after some tweaking, implement it. 
I would then review the code it wrote and continually refine it until it made sense to me. 
I originally tried letting Claude run wild but the problem is that it would get 90% of the way there (for more complicated tasks) and with each successive step the errors it made would get exponentially worse and worse. 

I've heard of people describing claude code as having a junior engineer but I think it's a very different relationship (irrespective of whether it can replace a junior engineer or not).
I wouldn't say that claude actually understands what im doing the same way a person would. 
If it did then it would probably fully replace me (and reduce a ton of my work). 
It knows things but also doesnt at the same time. 
I think there are some interesting agentic questions to solve here (or maybe it can be "solved" in the model architecture). 
The prime example is it writes me code, I notice architectural issues, I ask it to critique its own code, and it solves it and iterates. 

The smoothest part was asking Claude to write (quite comprehensive) unit tests and it was often able to debug them and make them pass in one shot. 
I also used Claude to debug random build issues (i.e. versioning).
This went less smoothly and Claude didn't make the more fundamental changes that would've fixed a recurring problem for good (i.e. installing the same version of Node in brew that electron uses). 

I found my experience to be similar to Edward Yang's post [here](https://blog.ezyang.com/) even though he used Codex w.r.t to UI. 
Specifically, I was able to create quite complex UIs with Claude code. 


