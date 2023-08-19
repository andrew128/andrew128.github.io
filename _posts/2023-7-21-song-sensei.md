---
layout: post
title: Song Sensei - Use an LLM for Song Suggestions
---

Visit [song-sensei.vercel.app](song-sensei.vercel.app) to chat with a bot llm to find the music you want on Spotify


## Post Outline
- [What and Why](#what-and-why)
- [Architecture](#architecture)
- [Key design decisions](#key-design-decisions)
- [What I learned](#what-i-learned)
- [Looking forward](#looking-forward)

## What and Why
Chat GPT has shown that it can find song suggestions based on conversational human input quite easily. The difference between this and spotify (and other song recommenders from my brief research) is the LLM component. The conversational format allows for the following sorts of questions:

- “I want a more dreamy and slow song suggestion”
- “I want instrumental suggestions”
- “Create a retro synth wave playlist for me”
- “Make it more edgy”
- “Make me a playlist of songs similar to x”

Of course these questions could all be done manually (e.g. by manually combing other song recommender outputs for instrumental only versions) but this app eliminates that work. In addition, it is connected to the Spotify API so the user can listen to song suggestions easily and create playlists automatically. 

## Architecture
Song Sensei combines the Spotify API and the Langchain API together in a React based web app written in NextJS and deployed on Vercel. 
This app consists of two pages. 
- song-sensei/pages/index.js: main page component that also performs sptoify authorization

- song-sensei/pages/chat/index.js: chat components that contains logic to automatically find and add search results to a new playlist in the user's logged in Spotify

## Key design decisions
- Making a website (as opposed to a script): more user friendly, accessible anywhere
- Model choice: Chat gpt 3.5 turbo because its the most complex version available publicly
- Hyperparam choice: Temperature of 0.8 because that was the recommended value in Stephen Wolfram’s blog post
- Spotify: I personally use spotify and am familiar with the interface
- NextJS/Vercel: I chose this stack because I was already familiar with React and the deployment was very easy for my use case

## What I learned
- how to set up open AI api - rate limiting, tokens
- Spotify api
- different models - da vinci vs chat gpt
- memory at a basic level, bufferered windows
- prompt engineering
- Next JS/Vercel deployments
- process.env and how to load environment variables
- Oauth authentication
- React concepts: state, router, useEffect

## Looking forward
- Add "memory" to model to be able to have more conversational mdeol

- Add song granularity: ability to choose whether want to add specific songs

- Add loading UI

- Add ability to play songs in browser

- Add streaming so that the output is streamed like messages and not sent all at once - also add a loading bar maybe

- use redux/react content for secure state management solutions instead of including tokens and access keys in the redirect URI query parameters

- Voice component: users can speak with the app in addition to texting

- Image background generation: users can converse with an image model to generate a picture for the playlist. Idea could be to first chat with bot to figure out songs, then chat with bot to figure out album cover. Could even ask bot to suggest cover art based on the following. 
