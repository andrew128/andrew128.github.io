---
layout: post
title: What is GraphRAG
---

A summary of a collection of (mostly Neo4j) resources on GraphRag. 

## Evolution from Basic RAG to GraphRAG
1. Traditional LLMs face challenges with factual accuracy
2. Vector-based RAG improves accuracy but lacks explanatory power
3. GraphRAG adds:
   - Higher accuracy and more complete answers
   - Easier maintenance and extensibility
   - Better explainability through graph visualization
   - More efficient token usage

## How does graph rag workflow work
Graph rag fundamentally just uses a knowledge graph that can optionally use a vector similarity component. 

![_config.yml]({{ site.baseurl }}/images/graphrag/fig1.png)

### Basic Example Graph RAG flow (lexical graph only)
Knowledge graph creation works like this:

1. **Content Processing**
   - Original content is split into manageable chunks
   - Each chunk is stored as a node in the graph database
   - Chunks maintain connections to preserve document structure

2. **Similarity Connections**
   - Highly similar chunks are connected using KNN (K-Nearest Neighbors)
   - These connections form a similarity-based sub-graph
   - Relationship type: "similar_to"

3. **Embedding Layer**
   - Embeddings are computed for each chunk
   - Embeddings are stored in:
     - The chunk nodes themselves
     - A separate vector index for efficient similarity search

4. **Entity Management**
   - Entities are extracted from chunks
   - Entities are stored as separate nodes
   - Connections are created between entities and their source chunks

Query processing works like this:

1. **Query Handling**
   - User question is received
   - Question is embedded into the same vector space

2. **Context Retrieval**
   - Most relevant chunks are retrieved from vector database
   - Retrieved based on embedding similarity to question

3. **Response Generation**
   - LLM receives structured input:
     - Original question
     - Retrieved context chunks
     - Relevant chat history
   - LLM generates response following guidelines:
     - Uses provided formatting
     - Includes source citations
     - Avoids speculation when information is unavailable

## GraphRAG Patterns
Some notes on [this blog post](
- https://neo4j.com/developer-blog/graphrag-field-guide-rag-patterns/).

### Graph Construction
1. **Domain Graphs**
   - Represent structured world knowledge
   - Can be created from structured or unstructured sources

2. **Lexical Graphs**
   - Created from unstructured text
   - Uses parsing and chunking strategies

3. **Combined Approaches**
   - Domain and lexical graphs can be integrated
   - Can use Neo4j knowledge graph builder
   - Storage options include separate databases or unified graph DB with vector support

### Basic Patterns (Lexical Graph)
1. **Basic Retriever**
   - Chunks text and creates embeddings
   - Uses vector similarity search for retrieval

2. **Parent-Child Retriever**
   - Splits content into parent and child chunks
   - Improves context by connecting related segments

3. **Hypothetical Question Retriever**
   - Generates potential questions for chunks
   - Improves matching between queries and content

### Intermediate Patterns (Domain Graph)
1. **Cypher Templates**
   - Uses predefined query templates
   - Matches user questions to existing templates

2. **Dynamic Cypher Generation**
   - Creates queries dynamically
   - More flexible than static templates

3. **Text2Cypher**
   - Translates natural language to queries
   - Most flexible but potentially less reliable

### Advanced Patterns (Combined Graphs)
1. **Graph-Enhanced Vector Search**
   - Combines vector search with graph traversal
   - Extracts entities and relationships
   - Provides richer context through connections

2. **Global Community Summary**
   - Forms hierarchical communities in graphs
   - Uses community-level summaries
   - Useful for global-scale questions

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