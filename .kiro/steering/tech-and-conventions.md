# Tech and Conventions

## Stack
- TypeScript, strict mode
- Vite + React for the web app
- Vitest for unit tests, fast-check for property-based tests
- No backend code until the spec calls for it

## Safety rules
- Generated text is never executed directly. It must parse into the
  spell AST first, or it is rejected.
- Spells run in a sandboxed interpreter with a step limit.
- Spells have no access to the DOM, network or storage.

## Testing
- Parser and interpreter changes require property-based tests.
- Every requirement maps to at least one test.

## Process
- Work from specs. Keep tasks small and traceable to requirements.
- Do not add features that are not in the current spec.