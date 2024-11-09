---
layout: post
title: What is GraphRAG
---

A summary of a collection of (mostly Neo4j) resources on GraphRag. 

## LLMS to Vector Rag to Graph Rag
- llms dont know truth
- vectgor based rag increases probability of correct answers but dont know truth and dont say why they chose something
- vector based rag valuable for when need to see how similar a group of words are to another
- dont get an "objective" truth
- knowledge graph represent the world, can represent truth here

- proven to be more accurate, use up less tokens
- can visualize responses, explainable
- summarize results from https://neo4j.com/blog/graphrag-manifesto/
    - higher accuracy, comre ocmplete answers
    - easiy to build and maintain RAG application - b/c graphs are easy to extend. (compared to vectors where its unclear how to continue expanding)
    - better explainability

## how does graph rag workflow work
- graph rag fundamentally just uses a knowledge graph that can optionally use a vector similarity component

- BASICALLY chunking documents and loading them into a vector database

- this data is then used to supplement the context for an llm response
<!-- INCLUDE FIG 1 screenshot -->

### graph construction

SOME TOOL HERE THAT CAN BE USED

- domain graphs - representation of the world (can be created from a structured source, unstructured, or both).
- lexical graphs - created from unstructured text. do this using parsing and chunking strategies.
- build a lexical unstructured knowledge graph using parsing and chunking strategies

- the above two can be combined

- use neo4j knowledge graph builder
- can store graph and vectors either separately in 2 distinct databses or use neo4j graph db which also supports vector search


### graph query
- do vector OR keyword search to find inital set of nodes. this can include just asking an LLM to construct cypher queries (agentigraph, neoconverse)
- traverse graph to bring back information about related nodes
- rerank documents using graph based ranking algorithm such as page rank OPTIONALLY

include image of graph rag pattern
image source: https://neo4j.com/blog/graphrag-manifesto/


## Example using neo4j llm knowledge grpah builder
https://neo4j.com/developer-blog/graphrag-llm-knowledge-graph-builder/

- creation:
- content split into chunks
- chunks are stored in graph with connections
- highly similar chunks are connected for KNN graph using similar relationship
- embeddings are computed and stored in chunks and vector index (embedding for each chunk I think)
- entities stored in graph and connected to originating chunks

- inputs and sources (question, vector results, chat history) used to generate response
- find most relevent chunks for a question and their connected entities up to depth of 2 hops. summarize chat chistory. send in customn prompt. llm is asked to provide and format response to question. prompt has formatting, citing soruces. not to speculate if answer is not known

#####################################################################

ABOVE IS DONE!!!!!!!!!!!!!!!!!!!!!!!!!!

FOCUS ON BELOW

## common patterns and areas of research
- https://neo4j.com/developer-blog/graphrag-field-guide-rag-patterns/
- https://graphr.ag/concepts/intro-to-graphrag/


1. pre retrieval - query rewriting, query entity extraction, query expansion (advanced)
2. actual retrieval
- performs vector search on specified node labels and properties. returns found nodes and similary scores as node and score which can be used in retrieval query to execute further traversals. 
- vector similarity search is used to return nodes with the best fit.
- so all of the nodes are somehow vectorized in a vector db and scored with indicies. 

3. post retrieval - rerank, summarization, fusion (advanced)
4. return response

- neoconverse: use llm to generate cypher query
- compare to graph rag which is about finding similar chunks in DB to LLM and then giving it all in a context to the LLM (see figure 1)
- QUESIOTN: maybe something about combining the two approaches? feedback where you can give human feedback to finding different chunks. or being able to specify certain areas. i guess it would have to do better than the existing nearest neighbors. but the benefit is being able to tell the LLM ...idk. one of the areas to explore. Talk to dad about this

- retrieval patterns on lexial graph (unstrctured data). so far seem to use vector dbs and rely on relationships present in unstructured data
- basic retriever:
- split unstructured text into smaller pieces and create embeddings for each of those pieces. 
- user's question is embedded using same embedded and performs vector similary search to retrieve k most similar chunks to add to context

- parent child retriever
- split documents into bbigger chunks and split those chunks into smaller chunks
- embed the child chunks
- user's question is used to find the most similar child chunks and then return the parent chunks that those child chunks are a part of. 
- useful when smaller chunks divide up the topics in a larger chunk better

- hypothetical question retriever
- use llm to generate hypotehtical questions answered  within chunks. questions are embedded. relationship between question and chunk stored

- retrieval patterns on domain graph. domain graph contains your business domain knowledge. 
- have cypher templates
- llm descries which one to use bsaed on user question
- this can be extended to dynamic cypher generation: generate cypher template and extract parameters from user query to pass in to cypher template
- user question translated to cypher query. llm can be provided with db description. not guaranteed to be able to translate. 

- retrieval patterns on lexical and domain graph combined:
- graph enhanced vector search::::;
- llm executes entity and relatoinship extraction on chunked unstructured text. the relationshpis are put in the grpah. i guess the extracted entities are the domain graph? domain graph is the entity and relationships I guess. yes confirmed from description. 
- global communtiy summary retriever:::::::::
- form hierarchical communities in domain graph
- user question and community level, community summaries are treive and given to llm. 
- communities created using leiden algorithm (graph algorithm for commiunty detection)
- lots of preprocessing required



## Further areas to explore
- retrieval strategies
- entity and relationship extraction from text
- graph algorithms
    - reranking documents using graph based ranking algorithm (after queries?)
- graph neural networks
- how FAISS and embeddings work. 

## Resources
- https://neo4j.com/developer-blog/graphrag-field-guide-rag-patterns/
- https://neo4j.com/blog/graphrag-manifesto/
- https://github.com/microsoft/graphrag
- https://neo4j.com/labs/genai-ecosystem/llm-graph-builder/: how to construct KG from unstructured text
- https://neo4j.com/developer-blog/genai-app-how-to-build/  (10/5/23)
- https://graphr.ag/concepts/intro-to-graphrag/ 
- Graph RAG survey paper: https://arxiv.org/pdf/2408.08921