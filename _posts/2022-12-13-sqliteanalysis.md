---
layout: post
title: How Does sqlite Work
---

## blah blah


-------------------------

personal notes::::::::::::::

## what is sqlite? and how does it relate to sql?

- C language library that implements a SQL database engine

- most used database engine in the world

- what is a sqlite database engine? core service for storing, processing, and securing data, provides controlled access and transaction processing to data I think
- interprets sql commands to access relational database - CRUD part to database )create, read update delete data
- OLTP vs OLAP - usages of sql db engines
- sql query translated to sql engine can process it
- storage engine writes to and retrives data from a data warehouse server
- query processor, query planner, etc. multiple steps involved
- https://www.heavy.ai/technical-glossary/sql-engine
- sql object is to manager large amounts of data especially with lots of data being written simultaneously, lots of data transactions
- sql is a specification - generalzied language
- rdbms is when the sql client communicates with the database how that data is managed during those actions
- 
- https://medium.com/@grepdennis/how-a-sql-database-engine-works-c67364e5cdfd

TODO: analyze the multiple steps involved in db engines, query processing, optimizations, maybelook at pavlos course, first process than optimize the query I think, then execute it using the dbms

## how can sqlite be used?
- https://engineering.fb.com/2020/03/02/data-infrastructure/messenger/

- https://www.sqlite.org/whentouse.html

## how does sqlite work? <- this will probably be more detailed as I come up with more questions

- https://www.sqlite.org/about.html <- high level description

- an in process library - WHAT IS IN PROCESS? 
- self contained
- serverless
- zero configuration
- transactional

- sqlite is an embedded sql database engine
- does not have a separate server process

- sqlite reads and writes directly to ordinary disk files
- complete sql databse with multiple tables, indices, triggers, views is contained in a single disk file

- database file format is cross platform - can freely copy db between 32 bit and 64 bit systems or big endian and little endian architectures

- the database file format: https://www.sqlite.org/fileformat2.html

- tradeoff between memory usage and speed?
- sqlite runs faster the more memory you give it - why exactly?
- faster than the filesystem - for reasons mentioned here: https://www.sqlite.org/fasterthanfs.html


- things that seem important:
- sqlite3 file object 
- vfs object
- tables, indexes, triggers, views

- https://www.sqlite.org/fullsql.html <- list of sqlite features

## disk io analysis/how it works
- default mechanism by which sqlite accesses and updates databse disk files is xRead and xWrite methods of the sqlite3_io_methods VFS object

- what is this vfs object? some kind of OS virtual file system object

- describes I/O performance: https://www.sqlite.org/fasterthanfs.html

## what kind of query planner does sqlite use?

## how does typing working in sqlite?
- sqlite uses dynamic typing, which means that the datatype of a value is associated with the value itself, not the container
- dynamic type system is backwards compatible with other db engines - sql statements that work on statically typed dbs work the same way in sqlite
- static typing, datatype of value is determine by its container (i.e. the particular column in which the value is stored)
- https://www.sqlite.org/datatype3.html


- one advantage of dynamic typing: https://www.sqlite.org/flextypegood.html
- flexible typing


- how does this remain performant? can't create indices on this i'm assuming?

## what are the advantages and disadvantages of sqlite compared to something else like postgres or mysql, (maybe even rocksdb? not really sure what that is)?

- sqlite is an embedded sql database engine
- does not have a separate server process
- can sqlite be used in a server setting?

- most sql db engines other than sqlite uses static rigid typing, https://www.sqlite.org/datatype3.html, sqlite uses dynamic typing


