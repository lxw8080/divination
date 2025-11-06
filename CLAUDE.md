# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**AI 算卦 (AI Divination)** is a Next.js application that performs traditional Chinese I Ching (Book of Changes) divinations using six coin tosses to generate hexagrams, then provides AI-powered interpretations using OpenAI models.

The application generates random hexagrams through coin toss animations, calculates the resulting I Ching hexagram based on traditional patterns, fetches classical interpretations from external sources, and uses AI to provide contextualized readings based on user questions.

## Development Commands

### Package Management
```bash
pnpm install          # Install dependencies
```

### Development
```bash
pnpm run dev         # Start development server with Turbo
```

### Build & Production
```bash
pnpm run build       # Build for production (standalone output)
pnpm run start       # Start production server
```

### Code Quality
```bash
pnpm run lint        # Run ESLint
```

## Environment Variables

Create `.env.local` for local development:

- `OPENAI_API_KEY`: OpenAI API key (required)
- `OPENAI_BASE_URL`: Custom API endpoint (defaults to `https://api.openai.com/v1`)
- `OPENAI_MODEL`: OpenAI model to use (defaults to `gpt-3.5-turbo`)

## Architecture

### App Structure (Next.js App Router)
- `app/page.tsx`: Main page component with Header, Divination, and Footer
- `app/layout.tsx`: Root layout with theme provider and Chinese font
- `app/server.ts`: Server action for AI interpretation using Vercel AI SDK
- `app/globals.css`: Global styles with Tailwind CSS

### Core Components (`components/`)
- `divination.tsx`: Main component orchestrating the entire divination flow
- `coin.tsx`: Animated coin flip component for generating random results
- `hexagram.tsx`: Visual hexagram display with 6 lines
- `question.tsx`: User question input component
- `result.tsx`: Displays hexagram result and interpretation
- `result-ai.tsx`: Streaming AI interpretation component
- `header.tsx` & `footer.tsx`: Layout components
- `ui/`: Reusable UI components (Button, Textarea, Dropdown Menu)

### Data Layer (`lib/`)
- `data/gua-index.json`: 8x8 matrix mapping upper/lower trigrams to hexagram numbers
- `data/gua-list.json`: Array of 64 hexagram names in Chinese
- `constant.ts`: Error prefix and version constants
- `utils.ts`: Utility functions using `clsx` and `tailwind-merge`
- `animate.ts`: Animation utilities for coin toss effects

### Key Algorithms

**Hexagram Calculation** (`components/divination.tsx:138-185`):
- Each coin toss generates a binary line (yang=true, yin=false)
- Six tosses create upper trigram (lines 3-6) and lower trigram (lines 0-2)
- Binary pattern maps to hexagram index using `gua-index.json` matrix
- Determines if lines are "changing" (0 or 3 heads = changing line)

**AI Interpretation** (`app/server.ts`):
- Fetches classical I Ching text from GitHub repository based on hexagram
- Constructs prompt with hexagram info, user question, and classical text
- Streams AI response with rate limiting for visual effect
- Error handling with prefixed error messages

### State Management
- All state managed in `divination.tsx` using React hooks
- No external state management library
- Server actions handle AI communication

### Styling & Theming
- Tailwind CSS with custom components
- `next-themes` for dark/light mode switching
- Chinese typography using LXGW WenKai Screen WebFont
- Responsive design with mobile-first approach

## External Dependencies

The application fetches classical I Ching interpretations from:
`https://raw.githubusercontent.com/sunls2/zhouyi/main/docs/{hexagramNumber}/index.md`

## Deployment

- Configured for Vercel deployment with standalone output
- Docker support included
- Analytics via Umami (optional component)