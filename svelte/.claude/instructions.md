---
applyTo: "**/*.svelte, **/*.ts, **/*.js, **/*.css"
---

# Global Instructions

## Language Policy

All instructions and prompts in this repository must be written in English.

## Development Workflow

When working with SvelteKit code, follow these instructions carefully.

### Implementation Steps

1. **Consult relevant skill files** and list which ones guided the implementation (e.g. `Skills used: [svelte5-sveltekit.md, tailwind-v4.md]`).

2. **Follow component-driven development**. Build small, reusable components before composing pages.

3. **Verify before committing**: Run `npm run check` and `npm run lint` to ensure no errors.
   Use `npm run dev` for local development with hot reload.

4. **Fix all TypeScript and ESLint errors** before proceeding.

### Path Conventions

SvelteKit uses file-based routing:

-   `src/routes/` - Pages and layouts
-   `src/lib/` - Shared components, utilities, server code
-   `src/lib/components/` - Reusable UI components
-   `src/lib/server/` - Server-only code (database, auth, etc.)

## Code Formatting

Prettier is configured with Svelte and Tailwind plugins.

```bash
# Format all files
npx prettier --write .

# Check only (CI)
npx prettier --check .

# Lint
npm run lint
```

-   Prettier handles all formatting automatically
-   ESLint configured with TypeScript and Svelte rules
-   Do not manually format code

## Svelte 5 Runes

Always use runes for reactivity:

-   `$state()` - Reactive local state
-   `$derived()` - Computed values
-   `$effect()` - Side effects with cleanup
-   `$props()` - Component props with TypeScript
-   `$bindable()` - Two-way binding

See [Svelte 5 & SvelteKit](./skills/svelte5-sveltekit.md) for detailed patterns.

## SvelteKit Conventions

-   Use `+page.svelte` for pages, `+layout.svelte` for layouts
-   Use `+page.server.ts` for server-side load functions and form actions
-   Use `+server.ts` for API endpoints
-   Use `+error.svelte` for error boundaries
-   Prefer `$lib` alias for imports from `src/lib/`
-   Use `use:enhance` for progressive form enhancement

## Tailwind CSS v4

This project uses Tailwind CSS v4. Key differences from v3:

-   Use `@import "tailwindcss"` instead of `@tailwind` directives
-   Use slash notation for opacity: `bg-blue-500/50` (not `bg-opacity-50`)
-   Use `shrink-*` and `grow-*` (not `flex-shrink-*`, `flex-grow-*`)
-   Use `gap-*` for spacing in flex/grid layouts
-   Always include `dark:` variants for theme support

See [Tailwind v4](./skills/tailwind-v4.md) and reference files in `skills/tailwind-v4-references/` for patterns.

## Styling Best Practices

-   Mobile-first responsive design (`text-sm md:text-base lg:text-lg`)
-   Always support dark mode with proper color pairings
-   Group utilities logically: layout → spacing → typography → colors → states
-   Include interactive states: `hover:`, `focus:`, `active:`, `disabled:`
-   Add `transition-colors` for smooth interactions
