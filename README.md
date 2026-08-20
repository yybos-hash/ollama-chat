**⚠️ This README was also AI-generated. I am definitely NOT putting some thought into this.**

# Ollama Chat

A small local web chat client for [Ollama](https://ollama.com). It talks directly to the Ollama HTTP API running on a machine on your local network (or `localhost`) — there's no backend of its own; it's just a static Svelte app that makes requests straight from your browser to your Ollama server.

## What it's for

Ollama exposes a REST API (`/api/chat`, `/api/tags`, `/api/ps`, `/api/generate`, etc.) for running local LLMs. This app is a lightweight, no-install UI on top of that API, useful when you're running Ollama headless on another box and don't want to use the terminal.

## Features

- Set the Ollama server address (`ip:port` or hostname) and pick a model from the ones it has pulled
- Streamed chat responses, including a collapsible panel for models that stream reasoning/`thinking` tokens separately
- Optional custom context size (`num_ctx`) sent per request
- A fixed system prompt telling the model to be brief and precise
- Stop generation mid-stream
- Unload a model from the server's memory on demand (`keep_alive: 0`)
- A live "loaded models" panel showing VRAM usage per model, polling `/api/ps` every few seconds

## Requirements

Your Ollama server needs to allow requests from this app's origin — set `OLLAMA_ORIGINS` on the machine running Ollama, otherwise the browser will block the requests with a CORS error.

## Running it

```
npm install
npm run dev
```

## Disclaimer

This app was vibe coded, built with an AI assistant rather than handwritten line by line. It hasn't been security audited or hardened. This was made purely for my own entertainment.
