# Implementation Plan: Grimoire Core

## Overview

This implementation plan breaks down the Grimoire Core feature into discrete coding tasks organized by architectural layer. The plan follows the three-layer separation (Domain → Persistence → UI) and prioritizes foundational components (parser, interpreter, data model) before building the UI layer.

**Implementation Language:** TypeScript (strict mode)

**Tech Stack:**
- TypeScript strict mode
- Vite + React for the web app
- Vitest for unit tests
- fast-check for property-based tests
- Zod for schema validation
- IndexedDB for storage

**Priority:** All tasks are [Core] unless marked with [Stretch]

---

## Tasks

### Phase 0: Project Automation

- [ ] 0.1 Create spell language steering file
  - Create `.kiro/steering/spell-language.md`
  - Document the complete Spell_Language grammar (copy from design.md)
  - Include examples of all statement types
  - Document reserved keywords
  - Document string escape sequences
  - _Requirements: 4.1, 4.2, 4.3_

- [ ] 0.2 Define development hooks
  - Create `.kiro/hooks/parser-tests.json`:
    - Trigger: PostFileSave
    - Matcher: `parser|grammar|tokenizer` (matches parser/grammar/tokenizer file paths)
    - Action: command to run parser tests and round-trip property tests
  - Create `.kiro/hooks/a11y-lint.json`:
    - Trigger: PostFileSave
    - Matcher: `ui/.*\.(tsx|jsx)` (matches UI component files)
    - Action: command to run eslint-plugin-jsx-a11y
  - Create `.kiro/hooks/requirements-check.json`:
    - Trigger: PostFileSave
    - Matcher: `requirements\.md|design\.md` (matches requirements/design files)
    - Action: command to verify requirement IDs cited in tasks.md exist in requirements.md
  - _Requirements: Development workflow_

### Phase 1: Project Setup and Foundation

- [ ] 1. Set up project structure and development environment
  - Initialize Vite project with React and TypeScript (strict mode)
  - Configure Vitest for unit testing
  - Configure fast-check for property-based testing
  - Install dependencies: Zod, React Testing Library, eslint-plugin-jsx-a11y
  - Set up ESLint and Prettier
  - Create directory structure: `src/domain/`, `src/persistence/`, `src/ui/`, `src/types/`
  - Configure build scripts and dev server
  - Set up development hooks (see Phase 0)
  - _Requirements: All (foundational)_

### Phase 2: Domain Layer - Data Models and Types

- [ ] 2. Define core TypeScript interfaces and types
  - [ ] 2.1 Create data model interfaces
    - Define `Spellbook`, `Card`, `CardObject` interfaces
    - Define `ButtonObject`, `TextFieldObject`, `LabelObject` interfaces
    - Define `CardSpell`, `SpellTarget`, `EventType` types
    - Define `SpellAST` and AST node types (see design.md AST Node Types)
    - Define `Token`, `ParseError`, `RuntimeError` types
    - Define `ExecutionResult`, `StateChange` types
    - Define `ValidationResult`, `ImportError`, `StorageError` types
    - _Requirements: 1.*, 2.*, 4.*, 5.*_

  - [ ] 2.2 Create utility types and constants
    - Define grammar constants (reserved keywords)
    - Define `FIELD_ID_REGEX` and validation constants
    - Define `Result<T, E>` type for error handling
    - Create UUID generation utility
    - _Requirements: 2.6, 4.2_

### Phase 3: Domain Layer - Spell Language Parser

- [ ] 3. Implement spell language tokenizer and parser
  - [ ] 3.1 Implement tokenizer
    - Write `tokenize(text: string): Token[]` function
    - Handle string literals with escape sequences (`\"`, `\\`, `\n`)
    - Track line and column numbers for error reporting
    - Recognize keywords (case-insensitive), identifiers, numbers, newlines
    - _Requirements: 4.1, 4.2, 4.3, 4.8_

  - [ ] 3.2 Implement recursive descent parser
    - Write `parseSpell(text: string): Result<SpellAST, ParseError>` function
    - Implement `parseStatement()`, `parseExpression()`, `parseCondition()` functions
    - Handle all statement types: `go to card`, `set`, `show message`, `if-then-else`
    - Generate plain-language error messages with line numbers
    - Validate Field_Identifiers against reserved keywords
    - _Requirements: 4.1, 4.2, 4.5, 4.7, 4.8, 4.9, 4.14_

  - [ ] 3.3 Write property test for parser round-trip
    - **Property 12: Spell Round-Trip Preserves Semantics**
    - **Validates: Requirements 4.7, 4.10**
    - Create fast-check arbitraries for SpellAST nodes
    - Generate random valid ASTs, print → parse → verify equivalence
    - Run 100+ iterations
    - _Requirements: 4.10_

  - [ ] 3.4 Implement spell pretty-printer
    - Write `printSpell(ast: SpellAST): string` function
    - Apply canonical formatting (lowercase keywords, 2-space indent)
    - Handle string escaping and quoting
    - _Requirements: 4.10_

  - [ ]* 3.5 Write unit tests for parser
    - Test each grammar rule (expressions, conditions, statements)
    - Test error conditions (unterminated string, invalid identifier, missing keyword)
    - Test edge cases (empty input, long input, special characters)
    - Verify error messages are plain language with line numbers
    - _Requirements: 4.1, 4.2, 4.3, 4.5, 4.8, 4.9_

  - [ ] 3.6 Write property test for invalid spell inputs
    - **Property 13: Invalid Spell Text Produces Parse Error**
    - **Validates: Requirements 4.5, 4.8, 4.11**
    - Generate random invalid spell text (token mutations)
    - Verify parsing fails gracefully (no crashes, returns ParseError)
    - _Requirements: 4.11_

### Phase 4: Domain Layer - Spell Interpreter

- [ ] 4. Implement spell interpreter and execution engine
  - [ ] 4.1 Implement expression and condition evaluation
    - Write `evaluateExpression(expr: Expression, fields: Map<string, string>): string`
    - Write `evaluateCondition(cond: Condition, fields: Map<string, string>): boolean`
    - Implement `is`, `is not`, `contains` operators (string-based comparisons)
    - _Requirements: 4.1, 4.12, 4.13, 5.6_

  - [ ] 4.2 Implement statement execution with step counting
    - Write `executeSpell(ast, card, cardId, cardNames, fields, stepLimit, navigationDepth): ExecutionResult`
    - Implement execution for each statement type
    - Track step counter and enforce step limit (10,000)
    - Return `StateChange[]` for successful execution
    - Maintain local field copy for within-spell reads/writes
    - Pass spellbook card names array to executeSpell for `go to card` validation
    - _Requirements: 4.1, 5.1, 5.2, 5.3, 5.6, 5.9_

  - [ ] 4.3 Implement runtime validation and error handling
    - Validate field references against card objects
    - Validate card targets against spellbook
    - Validate `go to card` targets against cardNames array (case-insensitive)
    - Generate plain-language error messages
    - Return `RuntimeError` with error statement reference
    - _Requirements: 2.10, 2.11, 4.14, 5.5_

  - [ ] 4.4 Implement navigation depth tracking
    - Enforce maximum navigation depth of 16
    - Return `NavigationDepthExceeded` error when limit reached
    - _Requirements: 5.7, 5.8_

  - [ ] 4.5 Write property test for interpreter determinism
    - **Property 14: Spell Execution is Deterministic**
    - **Validates: Requirements 5.6**
    - Generate random ASTs, card states, field values
    - Execute twice with identical inputs
    - Verify identical results (state changes, step count, errors)
    - _Requirements: 5.6_

  - [ ] 4.6 Write property test for step limit enforcement
    - **Property 15: Spell Step Limit Enforcement**
    - **Validates: Requirements 5.2, 5.3**
    - Generate random ASTs, calculate expected step count, test with limits from 1 up to the AST's step count
    - Verify execution halts at or before limit
    - Assert stepsExecuted <= stepLimit for all executions
    - Verify plain-language error message
    - _Requirements: 5.2, 5.3_

  - [ ] 4.7 Write property test for navigation depth limit
    - **Property 16: Navigation Depth Limit Enforcement**
    - **Validates: Requirements 5.7**
    - Generate navigation chains of varying depths
    - Verify execution halts at depth 16
    - Verify plain-language error message
    - _Requirements: 5.7_

  - [ ] 4.8 Write property test for field scope validation
    - **Property 17: Field Reference Scope Validation**
    - **Validates: Requirements 2.10, 2.11, 4.11, 4.14**
    - Generate spells with invalid field references
    - Verify runtime error with field name in message
    - _Requirements: 2.10, 2.11, 4.14_

  - [ ]* 4.9 Write unit tests for interpreter
    - Test each statement type execution
    - Test step counting accuracy
    - Test field scope validation
    - Test error message clarity
    - Test local field copy behavior (set then read within same spell)
    - _Requirements: 4.1, 5.1, 5.2, 5.5, 5.9_

  - [ ] 4.10 Write property test for interpreter totality
    - **Property 8: Interpreter Never Crashes**
    - **Validates: Reliability**
    - Generate random ASTs, verify execution always returns a result
    - Verify no uncaught exceptions
    - _Requirements: 5.1, 5.5_

- [ ] 4.11 Checkpoint: Interpreter Complete
  - Run all parser and interpreter tests
  - Verify all property-based tests pass with 100+ iterations
  - Ask the user if any questions or issues arise before proceeding to validation layer
  - _Requirements: 4.*, 5.*_

### Phase 5: Domain Layer - Validation

- [ ] 5. Implement validation functions
  - [ ] 5.1 Implement card name validation
    - Write `validateCardName(spellbook, cardIndex, newName): ValidationResult`
    - Enforce case-insensitive uniqueness within spellbook
    - Return plain-language error messages
    - _Requirements: 1.9, 1.10_

  - [ ] 5.2 Implement field identifier validation
    - Write `validateFieldId(fieldId): ValidationResult`
    - Check regex pattern (letters, digits, underscore; not starting with digit)
    - Check against reserved keywords
    - Write `validateFieldIdUnique(card, fieldId, excludeObjectId?): ValidationResult`
    - Enforce case-insensitive uniqueness within card
    - _Requirements: 2.6, 4.2_

  - [ ] 5.3 Implement spellbook validation
    - Write `validateSpellbook(spellbook): ValidationResult`
    - Check card name uniqueness
    - Check field ID uniqueness per card
    - Check spell target references exist
    - _Requirements: 1.9, 2.6, 7.12, 7.13_

  - [ ] 5.4 Write property tests for validation
    - **Property 6: Card Name Uniqueness**
    - **Validates: Requirements 1.9, 1.10**
    - Generate random spellbooks, test duplicate name rejection
    - **Property 11: Field_Identifier Uniqueness**
    - **Validates: Requirements 2.6**
    - Generate random cards, test duplicate field ID rejection
    - _Requirements: 1.9, 1.10, 2.6_

  - [ ]* 5.5 Write unit tests for validation
    - Test card name validation edge cases
    - Test field ID regex and keyword checks
    - Test spellbook validation completeness
    - _Requirements: 1.9, 1.10, 2.6, 7.12, 7.13_

### Phase 6: Persistence Layer - Storage

- [ ] 6. Implement IndexedDB storage adapter
  - [ ] 6.1 Implement database initialization
    - Write `openDatabase(): Promise<IDBDatabase>`
    - Create `spellbooks` object store with indices (by-modified, by-name)
    - Define schema version (1.0.0)
    - _Requirements: 7.1, 7.7_

  - [ ] 6.2 Implement save and load operations
    - Write `saveSpellbook(spellbook): Promise<void>`
    - Write `loadSpellbook(id): Promise<Spellbook>`
    - Write `loadAllSpellbooks(): Promise<Spellbook[]>`
    - Write `deleteSpellbook(id): Promise<void>`
    - Update `modifiedAt` timestamp on save
    - _Requirements: 7.1, 11.1_

  - [ ] 6.3 Implement export functionality
    - Write `exportSpellbook(spellbook): string`
    - Generate JSON in ExportFormat schema
    - Omit IDs for spellbook, preserve object IDs for references
    - Include schema version
    - _Requirements: 7.2, 7.11_

  - [ ] 6.4 Implement import validation and parsing
    - Write Zod schema for ExportFormat validation
    - Write `parseImport(fileContent): Result<SpellbookImportData, ImportError>`
    - Validate file size (max 5 MB)
    - Parse JSON and validate against schema
    - Check card name uniqueness (case-insensitive)
    - Check field ID uniqueness per card
    - Re-parse all spells and validate spell targets
    - Remap all object IDs to new UUIDs
    - _Requirements: 7.3, 7.4, 7.7, 7.8, 7.9, 7.10, 7.11, 7.12, 7.13_

  - [ ] 6.5 Implement import save operation
    - Write `saveImportedSpellbook(importData): Promise<Spellbook>`
    - Generate new spellbook ID and timestamps
    - _Requirements: 7.5_

  - [ ] 6.6 Write property test for export/import round-trip
    - **Property 18: Export/Import Round-Trip Preserves Structure**
    - **Validates: Requirements 7.6**
    - Generate random spellbooks
    - Export → import → verify structural equivalence
    - Verify name, card count, card names, object counts, spell counts match
    - _Requirements: 7.6_

  - [ ] 6.7 Write property test for import totality
    - **Property 9: Import Never Crashes**
    - **Validates: Reliability**
    - Generate random strings (including malformed JSON)
    - Verify parseImport always returns ok or error (no crashes)
    - _Requirements: 7.4_

  - [ ]* 6.8 Write unit tests for storage
    - Test save/load round-trip
    - Test export format correctness
    - Test import validation pipeline (each rejection case)
    - Test 5 MB size limit enforcement
    - Test spell re-parsing during import
    - Test ID remapping
    - Mock IndexedDB for storage failure scenarios
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.7, 7.8, 7.9, 7.10, 7.11_

  - [ ] 6.9 Implement storage error handling
    - Catch quota exceeded errors
    - Catch storage unavailable errors
    - Return structured StorageError with error kind and message
    - _Requirements: 10.1, 10.2, 10.3_

### Phase 7: Domain Layer - Spell Generation

- [ ] 7. Implement template-based spell generator [Core]
  - [ ] 7.1 Implement template fallback generator
    - Write `generateSpellFromTemplate(context): string`
    - Implement pattern matching for navigation, field setting, messages, conditionals
    - Extract card names, field IDs, quoted values using whole-word matching
    - Return valid spell text for all event types
    - _Requirements: 6.5, 6.6_

  - [ ] 7.2 Write property test for template generation
    - **Property 20: Template Generation Produces Valid Spells**
    - **Validates: Requirements 6.6**
    - Generate random contexts (event types, object types, fields, descriptions)
    - Verify generated spell text parses successfully
    - _Requirements: 6.6_

  - [ ]* 7.3 Write unit tests for template patterns
    - Test navigation pattern extraction
    - Test field setting pattern extraction
    - Test message pattern extraction
    - Test conditional pattern generation
    - Test keyword extraction helpers (whole-word matching)
    - _Requirements: 6.5, 6.6_

- [ ] 7.4 Checkpoint: Domain Layer Complete
  - Run all domain layer tests (parser, interpreter, validation, spell generation)
  - Verify no test failures
  - Ask the user if any questions or issues arise before proceeding to storage layer
  - _Requirements: 2.*, 3.*, 4.*, 5.*, 6.*, 7.*_

### Phase 8: UI Layer - State Management

- [ ] 8. Implement global application state
  - [ ] 8.1 Define state types and reducer actions
    - Create `AppState` interface
    - Define all reducer action types (see design.md State Reducer Actions)
    - _Requirements: All UI requirements_

  - [ ] 8.2 Implement state reducer
    - Write main reducer function with all action handlers
    - Enforce card name uniqueness in RENAME_CARD
    - Enforce field ID uniqueness in UPDATE_OBJECT
    - Call validation functions before state changes
    - Initialize runtime state in ENTER_BROWSE_MODE
    - Discard runtime state in EXIT_BROWSE_MODE
    - _Requirements: 1.*, 2.*, 3.*_

  - [ ] 8.3 Create React Context and provider
    - Create `AppContext` with state and dispatch
    - Write `AppProvider` component
    - Create `useAppState()` hook for components
    - _Requirements: All UI requirements_

  - [ ] 8.4 Write property test for reducer invariants
    - **Property: Reducer Maintains Invariants**
    - **Validates: State consistency**
    - Generate random initial states and action sequences
    - Apply actions to state
    - Verify invariants: unique card names, unique field IDs, valid spell targets
    - _Requirements: 1.9, 2.6_

  - [ ]* 8.5 Write unit tests for reducer
    - Test each action handler
    - Test validation enforcement (reject duplicate names, etc.)
    - Test runtime state lifecycle
    - _Requirements: 1.*, 2.*, 3.*_

### Phase 9: UI Layer - Core Components

- [ ] 9. Implement core UI components
  - [ ] 9.1 Create App shell component
    - Implement routing between StartScreen and SpellbookEditor
    - Set up AppProvider
    - Implement auto-save effect (debounced, 500ms)
    - _Requirements: 7.1, 11.1_

  - [ ] 9.2 Implement StartScreen component
    - Display user spellbooks and example spellbooks
    - Implement create new spellbook
    - Implement open spellbook
    - Implement rename spellbook with validation
    - Implement delete spellbook with confirmation
    - Visually distinguish user vs. example spellbooks
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 11.1, 11.2, 11.3, 11.4, 11.5, 11.6_

  - [ ] 9.3 Implement SpellbookEditor shell
    - Create editor layout with mode toggle, card navigator, card canvas
    - Handle mode transitions (Edit ↔ Browse)
    - Implement close spellbook (return to StartScreen)
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 3.1, 3.3, 3.6_

  - [ ] 9.4 Implement ModeToggle component
    - Display current mode (Edit | Browse)
    - Switch between modes on click
    - Dispatch ENTER_BROWSE_MODE / EXIT_BROWSE_MODE actions
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 3.1, 3.2, 3.3, 3.5_

  - [ ] 9.5 Implement CardNavigator component
    - Display current card position ("Card 2 of 5")
    - Implement previous/next navigation
    - Handle keyboard shortcuts (arrow keys when not in text input)
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 1.6_

- [ ] 9.6 Apply spooky-retro visual theme tokens
  - Define CSS custom properties for color palette
  - Apply high-contrast candlelit colors (--color-accent, --color-bg-dark, etc.)
  - Use Georgia for headings, Courier New for body
  - Add subtle glow effects on hover (not on focus)
  - Add parchment texture to cards (low contrast)
  - Ensure no effects obscure content
  - _Requirements: 9.1, 9.8_

### Phase 10: UI Layer - Card Canvas and Objects

- [ ] 10. Implement card canvas and object rendering
  - [ ] 10.1 Implement CardCanvas component
    - Render all objects on current card
    - Handle object selection in Edit mode
    - Display ObjectPalette in Edit mode
    - Coordinate drag/resize operations
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 2.1, 2.2, 2.3, 2.8_

  - [ ] 10.2 Implement ObjectRenderer components
    - Create ButtonObject renderer (label, click handler in Browse mode)
    - Create TextFieldObject renderer (input, field ID, default value)
    - Create LabelObject renderer (text display)
    - Apply Edit vs. Browse mode styling
    - Handle focus and keyboard navigation
    - Add accessible name (aria-label or visible label)
    - Add focus indicator styles (3px outline, high contrast)
    - _Requirements: 2.1, 2.5, 2.8, 4.5_

  - [ ] 10.3 Implement object placement (Edit mode)
    - Display ObjectPalette with Button/TextField/Label options
    - Handle drag-to-place workflow
    - Generate unique object IDs
    - Dispatch ADD_OBJECT action
    - _Requirements: 2.1, 2.7_

  - [ ] 10.4 Implement object move and resize (Edit mode)
    - Handle mouse drag for moving objects
    - Handle resize handles for resizing objects
    - Handle keyboard arrow keys for moving (when object selected)
    - Handle keyboard shift+arrow for resizing
    - Dispatch MOVE_OBJECT and RESIZE_OBJECT actions
    - _Requirements: 2.2, 2.3_

  - [ ] 10.5 Implement object editing (Edit mode)
    - Inline editing for Label text
    - Inline editing for Button label
    - Property panel for TextField (fieldId, defaultValue) with validation
    - Display field ID conflict errors and show spells that reference the old Field_Identifier when renaming (per Requirement 2.9)
    - Dispatch UPDATE_OBJECT action
    - _Requirements: 2.5, 2.6, 2.9_

  - [ ] 10.6 Implement object deletion (Edit mode)
    - Handle Delete/Backspace key when object selected
    - Confirm deletion for objects with attached spells
    - Dispatch DELETE_OBJECT action
    - _Requirements: 2.4_

### Phase 11: UI Layer - Spell Editor

- [ ] 11. Implement spell editor component
  - [ ] 11.1 Implement SpellEditor UI
    - Display plain-English input field
    - Display spell text editor (textarea with syntax highlighting if time permits)
    - Display Generate button
    - Display Confirm and Cancel buttons
    - Show parse errors inline with line numbers
    - _Requirements: 6.1, 6.3, 6.7_

  - [ ] 11.2 Implement spell generation flow
    - Collect generation context (event, object type, fields, card names, description)
    - Call generateSpellFromTemplate on Generate click
    - Parse generated spell text
    - Display candidate spell in editor (read-only or editable)
    - Show "not yet saved" indicator
    - _Requirements: 6.2, 6.4, 6.6_

  - [ ] 11.3 Implement spell editing and confirmation
    - Allow editing of candidate spell text
    - Re-parse on each keystroke
    - Display parse errors inline
    - On Confirm: parse final text, attach spell to object event
    - Dispatch ATTACH_SPELL action
    - _Requirements: 4.7, 6.3, 6.7_

  - [ ] 11.4 Open SpellEditor for object events
    - Right-click or button on objects to attach spell
    - Open editor for button click events
    - Open editor for card-open events (card-level)
    - _Requirements: 4.5, 4.6_

### Phase 12: UI Layer - Execution Controller

- [ ] 12. Implement spell execution controller
  - [ ] 12.1 Implement event-triggered execution
    - Handle button click events in Browse mode
    - Execute attached spell for button click
    - Apply StateChanges (field updates, messages, navigation)
    - Display error dialogs for runtime errors
    - Reset navigationDepth to 0 on user-initiated events
    - _Requirements: 4.5, 5.5, 5.8_

  - [ ] 12.2 Implement card-open spell execution
    - Fire card-open spell when navigating to a card in Browse mode
    - Fire card-open spell when entering Browse mode (depth 0)
    - Fire card-open spell after `go to card` navigation (depth + 1)
    - Enforce navigation depth limit of 16
    - _Requirements: 3.8, 4.6, 5.7, 5.8_

  - [ ] 12.3 Implement StateChange application
    - Apply SetField state changes to runtime field values
    - Apply ShowMessage state changes (display modal)
    - Apply NavigateCard state changes (switch card, fire card-open recursively)
    - Maintain within-spell field copy for deterministic reads
    - _Requirements: 4.1, 5.9_

  - [ ] 12.4 Implement error display
    - Display runtime errors in modal dialog
    - Display step limit exceeded message
    - Display navigation depth exceeded message
    - Highlight error statement in spell editor (Edit mode)
    - _Requirements: 5.3, 5.5, 5.7_

- [ ] 12.5 Checkpoint: Execution Controller Complete
  - Test spell execution in Browse mode (button clicks, card-open events)
  - Test navigation chains and depth limit
  - Test error handling (runtime errors, step limit, depth limit)
  - Ask the user if any questions or issues arise before proceeding to card management
  - _Requirements: 4.*, 5.*, 12.*_

### Phase 13: UI Layer - Card Management

- [ ] 13. Implement card management UI
  - [ ] 13.1 Implement add card
    - Add button to create new card
    - Dispatch ADD_CARD action with position
    - _Requirements: 1.2_

  - [ ] 13.2 Implement delete card
    - Delete button for current card
    - Confirm deletion if card has objects or spells
    - Reject deletion if last card (show error message)
    - Dispatch DELETE_CARD action
    - _Requirements: 1.3, 1.4_

  - [ ] 13.3 Implement reorder cards
    - Drag-and-drop to reorder cards
    - Keyboard controls for reordering (Ctrl+Up/Down when not in text input)
    - Dispatch REORDER_CARDS action
    - _Requirements: 1.5_

  - [ ] 13.4 Implement rename card
    - Inline editing for card name
    - Validate uniqueness (case-insensitive)
    - Display error message if duplicate
    - Dispatch RENAME_CARD action
    - _Requirements: 1.9, 1.10_

### Phase 14: UI Layer - Import/Export

- [ ] 14. Implement import and export UI
  - [ ] 14.1 Implement export spellbook
    - Export button in SpellbookEditor or StartScreen
    - Call exportSpellbook from storage module
    - Trigger browser download with `.grimoire.json` filename
    - _Requirements: 7.2, 10.3_

  - [ ] 14.2 Implement import spellbook
    - Import button on StartScreen
    - File input to select `.grimoire.json` file
    - Read file content (validate 5 MB limit before parsing)
    - Call parseImport and saveImportedSpellbook
    - Display import errors in modal dialog
    - Add imported spellbook to list
    - _Requirements: 7.3, 7.4, 7.5, 7.8, 7.10_

  - [ ] 14.3 Implement storage failure handling UI
    - Display warning banner on auto-save failure
    - Show quota exceeded message with export option
    - Show storage unavailable message with export option
    - Ensure export remains available when storage fails
    - _Requirements: 10.1, 10.2, 10.3_

### Phase 15: Accessibility and Styling

- [ ] 15. Implement accessibility features
  - [ ] 15.1 Implement keyboard navigation
    - Ensure all controls reachable by Tab
    - Set up logical tab order
    - Implement keyboard shortcuts (Escape, Enter, Space, arrows)
    - Disable shortcuts when focus in text input
    - _Requirements: 9.4_

  - [ ] 15.2 Implement focus indicators
    - Define visible focus indicator styles (3px outline)
    - Apply to all interactive elements
    - Ensure high contrast (WCAG AA)
    - _Requirements: 9.6_

  - [ ] 15.3 Implement ARIA labels
    - Add aria-label to all interactive controls
    - Add aria-live regions for announcements
    - Add aria-invalid and aria-describedby for validation errors
    - Add role attributes (button, region, dialog, alert)
    - _Requirements: 9.7_

  - [ ] 15.4 Implement screen reader announcements
    - Create announceToScreenReader utility
    - Announce spellbook saved, card navigation, mode switches
    - Announce error messages
    - _Requirements: 9.7_

  - [ ] 15.5 Implement reduced motion support
    - Detect prefers-reduced-motion media query
    - Create useReducedMotion hook
    - Provide manual toggle for reduced motion
    - Disable animations when reduced motion enabled
    - Apply reduced-motion class to disable CSS animations
    - _Requirements: 9.2, 9.3_

  - [ ]* 15.7 Verify WCAG AA contrast ratios
    - Write automated contrast checker utility
    - Test text contrast (4.5:1 minimum for normal text)
    - Test UI component contrast (3:1 minimum)
    - Run as part of test suite
    - _Requirements: 9.5_

  - [ ]* 15.8 Manual accessibility testing
    - Test keyboard navigation (all controls reachable)
    - Test screen reader with NVDA/JAWS
    - Test focus visibility
    - Test reduced motion
    - Cross-browser testing (Chrome, Firefox, Safari, Edge)
    - _Requirements: 9.4, 9.5, 9.6, 9.7_

- [ ] 15.9 Checkpoint: Accessibility Complete
  - Test keyboard navigation thoroughly
  - Test screen reader announcements
  - Test reduced motion mode
  - Verify WCAG AA contrast ratios
  - Ask the user if any questions or issues arise before proceeding to example spellbooks
  - _Requirements: 9.*_

### Phase 16: Example Spellbooks

- [ ] 16. Create example spellbooks
  - [ ] 16.1 Create Quiz example spellbook
    - Define spellbook with question card and complete card
    - Add label with question text (spooky theme)
    - Add text field for answer
    - Add submit button with conditional spell (if answer is correct, go to card)
    - Verify all spells parse successfully
    - _Requirements: 8.1, 8.4, 8.5_

  - [ ] 16.2 Create Shop Price List example spellbook
    - Define spellbook with products card and details card
    - Add labels for product names and prices (spooky theme)
    - Add button to view details (navigation spell)
    - Add back button on details card
    - Verify all spells parse successfully
    - _Requirements: 8.1, 8.4, 8.5_

  - [ ] 16.3 Create Flashcard Deck example spellbook
    - Define spellbook with multiple flashcard cards (spooky theme)
    - Add label with question text
    - Add "Show Answer" button (message spell)
    - Verify all spells parse successfully
    - _Requirements: 8.1, 8.4, 8.5_

  - [ ] 16.4 Implement example spellbook loading
    - Export example spellbooks as constants (EXAMPLE_SPELLBOOKS array)
    - Implement loadExampleSpellbook function (creates working copy)
    - Check for existing working copy before creating new one
    - Display examples on StartScreen
    - _Requirements: 8.2, 8.3_

  - [ ] 16.5 Implement startup example verification
    - Write verifyExampleSpellbooks function
    - Parse all example spells at startup
    - Log console errors if parsing fails
    - Do not prevent app from starting on failure
    - _Requirements: 8.6_

### Phase 17: Integration Testing

- [ ] 17. Write integration tests
  - [ ]* 17.1 Test mode transitions
    - Test Edit → Browse initializes runtime state
    - Test Browse → Edit discards runtime state
    - Test mode toggle UI updates
    - _Requirements: 3.2, 3.3, 3.7_

  - [ ]* 17.2 Test spell execution flow
    - Test button click triggers spell in Browse mode
    - Test card-open spell fires on navigation
    - Test StateChanges apply correctly (fields, messages, navigation)
    - Test error dialogs appear for runtime errors
    - _Requirements: 4.5, 4.6, 5.5_

  - [ ]* 17.3 Test storage operations
    - Test auto-save triggers after edits
    - Mock storage failure, verify warning appears
    - Test export produces downloadable file
    - Test import validates and loads spellbook
    - _Requirements: 7.1, 7.2, 7.3, 10.1_

  - [ ]* 17.4 Test card and object management
    - Test add/delete/reorder cards
    - Test place/move/resize/delete objects
    - Test validation errors display
    - _Requirements: 1.*, 2.*_

  - [ ]* 17.5 Test spell editor workflow
    - Test generate spell from description
    - Test edit candidate spell
    - Test parse errors display
    - Test confirm attaches spell
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.7_

### Phase 18: Polish and Documentation

- [ ] 18. Final polish and documentation
  - [ ] 18.1 Add loading states and feedback
    - Loading spinners for async operations
    - Success/error toast notifications
    - Debounced input validation feedback
    - _Requirements: User experience_

  - [ ] 18.2 Error message review
    - Audit all error messages for plain language
    - Ensure all errors include actionable guidance
    - Test error message clarity with non-programmers
    - _Requirements: 1.4, 1.10, 2.6, 4.8, 5.3, 5.5, 5.7, 7.4, 7.8, 10.1_

  - [ ] 18.3 Performance optimization
    - Optimize re-renders with React.memo
    - Debounce expensive operations (auto-save, validation)
    - Test with large spellbooks (100 cards)
    - _Requirements: Performance_

  - [ ] 18.4 Write user documentation
    - Create README with quick start guide
    - Document spell language grammar
    - Create example spellbook walkthroughs
    - Document keyboard shortcuts
    - _Requirements: User experience_

  - [ ] 18.5 Final end-to-end testing
    - Test complete user workflows (create → edit → test → export)
    - Test all example spellbooks
    - Cross-browser testing
    - Accessibility audit
    - _Requirements: All_

- [ ] 18.6 Deploy to static hosting
  - Build production bundle with Vite
  - Deploy to Netlify, Vercel, or GitHub Pages
  - Verify deployment from a clean browser (no dev tools, fresh IndexedDB)
  - Test all core workflows in production
  - _Requirements: Deployment_

- [ ] 18.7 Prepare submission artifacts
  - Create public GitHub repository
  - Add LICENSE file (MIT or Apache 2.0, clearly visible)
  - Commit `.kiro/` directory (do NOT add to .gitignore)
  - Write README with:
    - Project description and tagline
    - How to run locally (npm install, npm run dev)
    - Link to deployed demo
  - Create `KIRO_USAGE.md`:
    - Document how Kiro was used in this project
    - Note which Kiro features were most helpful
    - Include any steering files or hooks created
  - Write demo video script (under 3 minutes):
    - Show creating a spellbook
    - Show placing objects and writing a spell
    - Show testing in Browse mode
    - Show export/import
  - _Requirements: Submission_

### Phase 19: AI Service Integration [Stretch]

- [ ] 19.1 Implement AI service endpoint
  - Create server endpoint (Node.js/Express or serverless function)
  - Integrate with AI provider (OpenAI, Anthropic, or similar)
  - Store API key in environment variable (never in client code)
  - Implement input validation (description length ≤ 500 chars, fields ≤ 20, cards ≤ 100)
  - Implement required rate limiting (10 requests per minute per IP)
  - Return spell text in JSON response
  - _Requirements: 12.1, 12.2_

- [ ] 19.2 Implement AI service client
  - Write `generateSpellWithAI(description, context, options)` function
  - Display privacy notice before first request (description sent to server)
  - Enforce 10-second timeout
  - Fall back to Template_Fallback on timeout or error
  - Handle network errors gracefully
  - _Requirements: 12.2, 12.3_

- [ ] 19.3 Document AI service endpoint for judges
  - Add endpoint URL to README
  - Document that the endpoint can be used without an API key (public demo endpoint)
  - Or document that Template_Fallback is used when endpoint is unavailable
  - _Requirements: Submission_

- [ ]* 19.4 Test AI service integration
  - Test privacy notice displays
  - Test timeout handling
  - Test fallback to Template_Fallback
  - Mock AI service responses for testing
  - _Requirements: 12.*_

---

## Notes

- **Tasks marked with `*` are optional** (testing tasks) and can be skipped for faster MVP, but are strongly recommended for correctness.
- Core implementation tasks (not marked with `*`) must be completed.
- Property-based tests validate universal correctness properties and should be prioritized.
- Unit tests validate specific examples and edge cases.
- Integration tests validate UI workflows and component interactions.
- Each task references specific requirements for traceability.
- The dependency graph below shows the execution order for parallel task scheduling.

---

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["0.1", "0.2"] },
    { "id": 1, "tasks": ["1"] },
    { "id": 2, "tasks": ["2.1", "2.2"] },
    { "id": 3, "tasks": ["3.1"] },
    { "id": 4, "tasks": ["3.2", "3.4"] },
    { "id": 5, "tasks": ["3.3", "3.5", "3.6"] },
    { "id": 6, "tasks": ["4.1"] },
    { "id": 7, "tasks": ["4.2", "4.3", "4.4"] },
    { "id": 8, "tasks": ["4.5", "4.6", "4.7", "4.8", "4.9", "4.10"] },
    { "id": 9, "tasks": ["4.11"] },
    { "id": 10, "tasks": ["5.1", "5.2", "5.3"] },
    { "id": 11, "tasks": ["5.4", "5.5"] },
    { "id": 12, "tasks": ["6.1"] },
    { "id": 13, "tasks": ["6.2", "6.3", "6.4", "6.5"] },
    { "id": 14, "tasks": ["6.6", "6.7", "6.8", "6.9"] },
    { "id": 15, "tasks": ["7.1"] },
    { "id": 16, "tasks": ["7.2", "7.3"] },
    { "id": 17, "tasks": ["7.4"] },
    { "id": 18, "tasks": ["8.1", "8.2", "8.3"] },
    { "id": 19, "tasks": ["8.4", "8.5"] },
    { "id": 20, "tasks": ["9.1"] },
    { "id": 21, "tasks": ["9.2", "9.3", "9.4", "9.5"] },
    { "id": 22, "tasks": ["9.6"] },
    { "id": 23, "tasks": ["10.1", "10.2"] },
    { "id": 24, "tasks": ["10.3", "10.4", "10.5", "10.6"] },
    { "id": 25, "tasks": ["11.1", "11.2", "11.3", "11.4"] },
    { "id": 26, "tasks": ["12.1", "12.2", "12.3", "12.4"] },
    { "id": 27, "tasks": ["12.5"] },
    { "id": 28, "tasks": ["13.1", "13.2", "13.3", "13.4"] },
    { "id": 29, "tasks": ["14.1", "14.2", "14.3"] },
    { "id": 30, "tasks": ["15.1", "15.2", "15.3", "15.4", "15.5"] },
    { "id": 31, "tasks": ["15.7", "15.8"] },
    { "id": 32, "tasks": ["15.9"] },
    { "id": 33, "tasks": ["16.1", "16.2", "16.3"] },
    { "id": 34, "tasks": ["16.4", "16.5"] },
    { "id": 35, "tasks": ["17.1", "17.2", "17.3", "17.4", "17.5"] },
    { "id": 36, "tasks": ["18.1", "18.2", "18.3", "18.4"] },
    { "id": 37, "tasks": ["18.5"] },
    { "id": 38, "tasks": ["18.6"] },
    { "id": 39, "tasks": ["18.7"] },
    { "id": 40, "tasks": ["19.1"] },
    { "id": 41, "tasks": ["19.2", "19.3"] },
    { "id": 42, "tasks": ["19.4"] }
  ]
}
```
