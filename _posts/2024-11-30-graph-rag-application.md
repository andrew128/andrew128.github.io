---
layout: post
title: Simple Graph Rag Application
---
A description of a simple graph rag application. 

## Problem
The way I think is in terms of connections and entities which seem to be a natural thought pattern in humans. 
The use case I want to solve for is having lots of ideas and throwing them in a list structure but not having a good way of exploring, surfacing, or drawing connections between my ideas.

## Solution
The basic solution I have in mind is a graph rag application.
The idea is to be able to “upload” ideas and thoughts in the form of unstructured text through a natural language interface.
The other user action is being able to ask questions and query information about the built up knowledge graph. 
This will require embedding the user question and doing a vector search on the unstructured text nodes in the KG.

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
- See https://github.com/andrew128/simple-graph-rag-app for the code repo