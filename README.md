# Relay Map

Research prototype for **Relay**: an interactive support constellation that helps someone facing a difficult situation see the people, places, tools, and first steps that can help them move forward.

This repo is an individual Research Milestone sample app. It explores whether SvelteKit is a good fit for Relay’s interactive map UI.

## What you can do

- Select bubbles to inspect details and a suggested next action
- Add a custom support bubble (label, type, details, next action)
- New bubbles connect automatically to the central situation
- Delete custom bubbles (seeded demo nodes stay protected)
- Drag bubbles to rearrange the constellation
- Save chosen steps in a **My next actions** list
- Persist the constellation and next actions in `localStorage` across refresh

## Tech stack

| Piece | Choice |
| --- | --- |
| Framework | [SvelteKit](https://svelte.dev/docs/kit) + Svelte 5 |
| Language | TypeScript |
| Bundler / dev server | Vite |
| Package manager | npm |
| Persistence | Browser `localStorage` (no backend) |

### Why SvelteKit for this prototype

- Reactive UI with little boilerplate — useful for a map that updates as users add, drag, and delete nodes
- Svelte 5 runes (`$state`, `$derived`, `$effect`) keep constellation state easy to follow
- Fast local feedback with Vite, which matches a research/spike workflow
- Adapter-based deploy story later if the team wants to host a demo

Tradeoff: this sample is client-only. A production Relay app would still need a real backend for accounts, shared Circles data, and sync across devices.

## Run locally

Requirements: Node.js 20+ and npm.

```sh
npm install
npm run dev
```

Open the URL Vite prints (usually `http://localhost:5173`).

Other scripts:

```sh
npm run check    # TypeScript + Svelte diagnostics
npm run build    # production build
npm run preview  # preview the production build
```

## Project layout

```text
src/routes/+page.svelte   # constellation UI, form, persistence, interactions
src/routes/+layout.svelte # shared shell / favicon
src/lib/                  # shared assets
```

## Notes for the team

This is a front-end spike, not the final Relay product. It is meant to prove:

1. Users can interact with a situation map (not a static diagram)
2. Adding personal supports feels concrete and controllable
3. SvelteKit is productive enough for this kind of UI

After the milestone, compare findings with teammates before locking the stack for the team project.
