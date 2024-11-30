---
layout: post
title: Simple Graph Rag Application
---
A description of a simple graph rag application. 

## Problem
The way I think is in terms of connections and graphs. Its probably how other people think too. I think it works especially well for me. I always have lots of ideas of things that I want to work on but I throw it in different notes in the notes app and dont have a good way of exploring or surfacing my ideas. 

## Solution
App that stores a knowledge graph of your thoughts and notes with a natural language interface using LLMs. this is a graph rag application if we’re going buzzwordy. The idea is to be able to “upload” ideas and thoughts in the form of unstructured text through a natural language interface. This is one user action. 

There will be some parsing and chunking done here. That will then store the unstructured text in the KG. 

The other user action is being able to ask questions and query information about the built up knowledge graph. This will require embedding the user question and doing a vector search on the unstructured text nodes in the KG.

Similarly I want to be able to view the knowledge graph and interact with it in a UI. This should come for free with the inherent graph structure though (although obviously the UI needs to be built).  

## MVP Design
This architecture is the the simplest possible graph rag design that is designed well to scale and handle new graph rag architectures. I’m going to start off with a simple lexical unstructured graph structure. 
The architecture is a simple lexical unstructured graph structure. 

The current client code has a react front end that displays a page with two tabs: Add Knowledge and Query Knowledge. 
Each of these tabs uses the two components InputForm and ResponseDisplay. 
The client has two functions addNode and queryGraph that correspond to Add Knowledge and Query Knowledge that make POST request calls to the server side. 
This is all wrapped in a node app.

The server side is an express node app that defines the two endpoints /nodes and /query. /nodes is where we call addNode. /quey calls queryGraph()

addNode generates an embedding using open ai embeddings api and then stores it in neo4j.
Below is the cypher query that is run.
```
const result = await session.run(
`
CREATE (n:TextNode {
        text: $text,
        embedding: $embedding,
        createdAt: datetime()
    })
    RETURN n
    `,
    { text, embedding }
);
```

queryGraph generates an embedding from input and queries from neo4j with the following query. 
It gets the 3 most similar chunks. 
```
const result = await session.run(
    `
    MATCH (n:TextNode)
    WITH n, gds.similarity.cosine(n.embedding, $embedding) AS similarity
    ORDER BY similarity DESC
    LIMIT 3
    RETURN n, similarity
    `,
    { embedding }
);
```

## Productionization
I chose to only locally test and not deploy this anywhere because I'm interested in exploring more complicated graph rag algorithms and architectures first.
This app is a proof of concept that I may use as a template to come back to later.

There are several gaps that would have to be covered before productionizing:
- security hardening (using certificates, https, etc.)
- deploy client side (maybe Vercel to start off with)
- deploy server side (maybe Railway.app or Heroku)
- deploy neo4j somewhere (probably Aura)

## Some ideas for future work
- more complicated and interesting graph rag algorithms
    - no chunking or parsing is done here for example
- benchmarks and eval sets to understand performance
- fancy graph visualization UI
- better project structure
    - in the future I'd probably use next.js to automatically create a project structure

## Resources
- Graph RAG survey paper: https://arxiv.org/abs/2408.08921 