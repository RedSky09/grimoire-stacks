# Grimoire

**Build small apps from cards, in plain English.**

Grimoire is a browser-based tool where non-programmers build small
interactive apps from stacks of cards ("spellbooks"). Place buttons, text
fields and labels on cards, then attach short English-like scripts
("spells") that make them interactive. Describe what you want in plain
English and Grimoire proposes the spell for you to review.

Built for the **Kiroween 2026** hackathon, **Resurrection** category: it
revives the card-and-stack idea of 1980s-90s authoring tools for today.
It is inspired by those tools and uses no code, assets or names from
them. It is not affiliated with other projects named Grimoire.

> **Status:** in development. See "Features" for what works today.

## Features

- [ ] Spellbooks made of named cards (add, delete, reorder, rename)
- [ ] Buttons, text fields and labels (place, move, resize, edit)
- [ ] Edit mode and Browse mode
- [ ] Spell language with a sandboxed, step-limited interpreter
- [ ] Plain-English to spell generation (template-based; AI-assisted
      generation is a planned extra)
- [ ] Automatic saving in the browser, plus export and import
- [ ] Three example spellbooks
- [ ] Keyboard-friendly and screen-reader-friendly, with a reduced-motion mode

## The spell language (preview)

```
if field answer is "4" then
  show message "Correct!"
  go to card "Complete"
else
  show message "Try again"
end if
```

Spells are parsed into a syntax tree and run by a sandboxed interpreter.
Spell text is never evaluated as code. The syntax may change until the
parser is finished; see `.kiro/steering/spell-language.md`.

## Quick start

Requires Node.js (version to be confirmed).

    npm install
    npm run dev      # start the app
    npm test         # run unit and property-based tests

## How it was built with Kiro

Grimoire was built spec-first with Kiro. The `.kiro/` directory contains:

- `specs/grimoire-core/`: requirements, design and task list
- `steering/`: product, style and language rules that guide Kiro's output
- `hooks/`: automations (parser tests, accessibility lint, spec checks)

A fuller write-up is in `KIRO_USAGE.md` (to be added).

## License

MIT. See `LICENSE`.