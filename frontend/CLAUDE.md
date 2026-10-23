# frontend — web client

Read the root [CLAUDE.md](../CLAUDE.md) first for repo-wide rules.

## Responsibility

- The browser-based client for the Family app.
- Gets all data through the backend HTTP API. It holds no business rules that the backend does not also enforce.

## Stack

- Framework: Vue 3 with the Composition API.
- Build tool: Vite, the default from `create-vue`.
- Routing: Vue Router.
- State: Pinia.
- Language: not decided. TypeScript is the default choice unless decided otherwise.
- Styling approach: not decided.
- Test framework: not decided. Vitest with Vue Test Utils is the default choice unless decided otherwise.
- Package manager: not decided. npm is assumed below.

## Commands

Run from `frontend/`. Confirm these once the project is scaffolded.

- Install: `npm install`
- Run dev server: `npm run dev`
- Build: `npm run build`
- Preview production build: `npm run preview`
- Unit tests: `npm run test:unit`
- Lint: `npm run lint`

## Conventions

- Use the Composition API with `<script setup>` in every component. Do not use the Options API.
- Put reusable stateful logic in composables named `useXxx` under `src/composables/`.
- Declare props with `defineProps` and events with `defineEmits`, typed when using TypeScript.
- Use Pinia stores defined in setup style for state shared across pages. Keep component-local state in `ref` and `reactive`.
- Name components in PascalCase, one component per `.vue` file.

## Choosing a family

A user can belong to several families. They choose which one they are managing.

- After login, show a family picker listing the user's families from `/api/me/families`.
- The profile menu has a family switcher available on every page.
- If the user has exactly one family, open it directly without showing the picker.
- If the user has no families yet, show onboarding to create a family or accept an invitation.
- Remember the last chosen family on this device and open it on the next launch, as long as the user is still a member.
- If the backend returns 404 for the chosen family, the membership has ended. Remove that family from the switcher and show the picker again.
- Put the chosen family ID in the URL, such as `/families/:familyId/...`. Links then point to the right family, and two tabs can show two different families.
- Keep the user's family list in a Pinia store. The API layer takes the family ID from the current route, never from a global variable.

## Rules

- Keep API calls in one dedicated layer under `src/api/` instead of scattering requests across components.
- The backend API base URL comes from a `VITE_` environment variable, never hard-coded.
- Never put secrets in frontend code or `VITE_` variables. Anything shipped to the browser is public.
- Health data, as defined in the backend's "Health data encryption" section (body sizes, feeding, sleep, medical examinations and more), is medical data. Keep it in memory only: never in `localStorage`, `sessionStorage`, IndexedDB, URLs or query strings, or console logs. Clear it from Pinia stores on logout and when switching families.
- Pages must work at phone width as well as desktop.
- Keep shared UI pieces as reusable components rather than copy-pasting markup.
