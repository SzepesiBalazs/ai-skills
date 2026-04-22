---
name: gemini-chatbot
description: "Add a free Gemini chatbot interface to a Vue 3 app. Use when: building a chatbot UI, creating an AI chat panel, adding a conversational AI assistant, integrating Google's Gemini models for free."
argument-hint: "The purpose or persona of the chatbot (e.g. general assistant, coding helper)"
---

# Gemini Chatbot

## When to Use

- Adding a live AI chat panel backed by Google's Gemini API (free Standard tier)
- Teaching users how to integrate LLM APIs in a Vue 3 app
- Demonstrating streaming SSE responses and conversation history management

## Models to Use

All models below are **free of charge** on the Standard tier (rate-limited).
Get a free API key at https://aistudio.google.com/apikey.

| Model                   | Notes                          |
| ----------------------- | ------------------------------ |
| `gemini-2.5-flash-lite` | Default — fastest, cheapest    |
| `gemini-2.5-flash`      | Balanced performance           |
| `gemini-2.5-pro`        | Highest quality, free Standard |

## Architecture

| File                               | Purpose                                    |
| ---------------------------------- | ------------------------------------------ |
| `src/views/ChatbotView.vue`        | Page layout — wraps ChatbotPanel           |
| `src/components/ChatbotPanel.vue`  | Chat UI: message list, input, model picker |
| `src/composables/useChatbot.ts`    | Gemini fetch + streaming + state           |
| `src/router/index.ts` + `index.js` | Register `/chatbot` route                  |
| `src/App.vue`                      | Add nav link                               |

## API Key Handling

- User pastes their Google AI Studio key (starts with `AIza...`) into a dedicated input
- It is saved to `localStorage` via `apiKey` ref under key `gemini-api-key`
- The key is NEVER hardcoded or committed
- Uses Gemini's OpenAI-compatible endpoint:
  `https://generativelanguage.googleapis.com/v1beta/openai/chat/completions`

## Streaming

Use `stream: true` in the request body. The Gemini OpenAI-compatibility endpoint
returns the same SSE format as the OpenAI API. Read chunks from the
`ReadableStream` response body, decode each SSE line (`data: {...}`), and append
`choices[0].delta.content` to the last assistant message in `messages`.

## Composable Shape

```ts
// src/composables/useChatbot.ts
const { messages, send, isLoading, error, apiKey, selectedModel, clearChat } =
  useChatbot();
```

- `messages` — `Ref<Message[]>` where `Message = { role: 'user'|'assistant', content: string }`
- `send(text: string)` — pushes user message, streams assistant reply
- `isLoading` — `Ref<boolean>` true while streaming
- `error` — `Ref<string>` set if the request fails
- `apiKey` — `Ref<string>` persisted to localStorage
- `selectedModel` — `Ref<string>` defaults to `'gpt-4.1-nano'`
- `clearChat` — resets `messages` to empty

## Component Conventions

- Use `<script lang="ts">` with `setup()` — NOT `<script setup>`
- Tailwind CSS only — no `<style>` blocks
- User messages right-aligned; assistant messages left-aligned
- Show a pulsing dot indicator while `isLoading` is true
- Show `error` in a red banner if set
