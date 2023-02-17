---
layout: post
title: Web App To Download Saved Reddit Posts
---

tldr visit [this link](https://savedredditpostextensionmain.gatsbyjs.io/) to download your saved reddit posts.

## Outline
- [Problem](#Problem)
- [Solution](#Solution)
- [Milestones](#milestones)
- [Learnings](#learnings)
- [Future work](#future-work)

## Problem
Reddit only saves up to ~1000 reddit posts before truncating them.

## Solution
Use the [Reddit API](https://www.reddit.com/dev/api/) to download all the saved posts.

The overall design is to have a simple `index.js` homepage that displays the authorization link to redirect users to so as to ask for permission to access their saved history.
There is also a React component implemented as an es6 class that uses Axios to fetch the saved post data using reddit's listing technique (i.e. paging through data) until there is no more data to be downloaded.
When the button is clicked, a handler downloads the data as a JSON file.

## Milestones

### 1: Figure out the stack
- decide what kind of app I want to use (web app, script, etc.)
- decide deployment strategy (gatsby cloud, github pages, etc.)
- figure out stack (gatsby, react, axios, etc.)

### 2: Connect with Reddit API
- Grok the [reddit oauth api](https://github.com/reddit-archive/reddit/wiki/OAuth2#application-only-oauth)
- implement authorization code

### 3: Fetch the data
- Understand the [reddit api for fetching saved posts](https://www.reddit.com/dev/api/#GET_user_{username}_saved)
- implement fetch code

### 4: Download the data
- implement button to download fetched data

## Design Decisions Made

### User flow
I’ve decided the only rendering to be done is through the saved file format taken from the server. I’ve decided against adding rendering of a post directly from reddit (extra work not needed for mvp). 

### Use of Static Sites
Going with a static site generator to generate a static site because the only work needing to be done is querying the data from the reddit api which can be rendered in a template and served back to the user’s browser. The user does not provide much varied input at all (only requesting all the saved posts). Maybe in the future there will be more user options but that is not needed for mvp. Even with metrics those can be precomputed on the server I think. (maybe cached too).

Static pages eliminate the latency that databases introduce when the server has to get all the data from a database, merge with template files and generate html page as a response. 

### Deployment on Gatsby cloud
This is a simple web app and I was already using gatsby.

### Use of Gatsby JS
I was interested in learning how to make a web app and gatsby seemed like an interesting library to learn and play with.

## Learnings
- Gatsby development
- Github pages
- Deployment in gatsby
- Static pages
- react/jsx, React components, lifecycle functions, es6 class
- Node package installation
- Authorization and security, client secret (oauth)
- Cors and http headers
- Javascript syntax and language (promise chains, async await, arrow functions, etc.)

## Future Work
- better UI
- can sort by subreddit
- search text of posts
- add refresh token button
- display stats on saved posts (e.g. timeline of how many saved posts in what subreddit)
- display downloaded data, sorted by date
- use count in reddit parameter for getting saved output listings
