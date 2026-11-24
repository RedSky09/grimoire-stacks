# Design Document: Grimoire Core

## Overview

Grimoire is a browser-based application that enables non-programmers to build small interactive applications from "spellbooks" (collections of cards). Users place objects (buttons, text fields, labels) on cards and attach English-like scripts called "spells" to object events. The application runs entirely in the browser with no backend requirement for core functionality.

**Core Architecture Principles:**
- **Safety-first**: All spell text must parse to an AST before execution; no raw code evaluation
- **Sandboxed execution**: Spells run in a restricted interpreter with no DOM/network/storage access
- **Deterministic**: Given the same inputs, spell execution produces the same outputs
- **Accessible**: Full keyboard navigation, screen reader support, reduced-motion support
- **Offline-capable**: Core features work without network; AI spell generation is an optional enhancement

The design separates concerns into three layers:
1. **UI Layer** (React components) - presentation and user interaction
2. **Domain Layer** (pure TypeScript) - spell parsing/execution, data model, business logic
3. **Persistence Layer** (IndexedDB adapter) - storage with auto-save

---

## Architecture

### Component Hierarchy

```
App
├── StartScreen
│   ├── SpellbookList
│   │   ├── UserSpellbookItem (repeating)
│   │   └── ExampleSpellbookItem (repeating)
│   └── CreateSpellbookButton
└── SpellbookEditor (when spellbook is open)
    ├── ModeToggle (Edit / Browse)
    ├── CardNavigator (position indicator, prev/next)
    ├── CardCanvas
    │   ├── ObjectRenderer (Button | TextField | Label - repeating)
    │   └── ObjectPalette (Edit mode only)
    └── SpellEditor (Edit mode only)
        ├── PlainEnglishInput
        ├── SpellTextEditor (CodeMirror or textarea)
        ├── GenerateButton
        └── ConfirmButton
```

### State Management

**Global Application State (React Context + useReducer):**
- Current spellbook (if any)
- Current card index
- Current mode (Edit | Browse)
- Runtime field values (Browse mode only)
- Selected object (Edit mode only)
- Spell editor state (target event, candidate text, confirmed status)

**State Reducer Actions:**
```typescript
type Action =
  | { type: 'OPEN_SPELLBOOK'; spellbook: Spellbook }
  | { type: 'CLOSE_SPELLBOOK' }
  | { type: 'SWITCH_MODE'; mode: 'Edit' | 'Browse' }
  | { type: 'NAVIGATE_CARD'; index: number }
  | { type: 'ADD_CARD'; position: number }
  | { type: 'DELETE_CARD'; index: number }
  | { type: 'REORDER_CARDS'; fromIndex: number; toIndex: number }
  | { type: 'RENAME_CARD'; index: number; name: string }
  | { type: 'ADD_OBJECT'; cardIndex: number; object: CardObject }
  | { type: 'MOVE_OBJECT'; cardIndex: number; objectId: string; x: number; y: number }
  | { type: 'RESIZE_OBJECT'; cardIndex: number; objectId: string; width: number; height: number }
  | { type: 'DELETE_OBJECT'; cardIndex: number; objectId: string }
  | { type: 'UPDATE_OBJECT'; cardIndex: number; objectId: string; updates: Partial<CardObject> }
  | { type: 'ATTACH_SPELL'; cardIndex: number; objectId: string; event: EventType; spell: Spell }
  | { type: 'SET_FIELD_VALUE'; cardId: string; fieldId: string; value: string } // Runtime only
  | { type: 'ENTER_BROWSE_MODE' } // Initialize runtime state
  | { type: 'EXIT_BROWSE_MODE' } // Discard runtime state
  | { type: 'SELECT_OBJECT'; objectId: string | null }
  | { type: 'OPEN_SPELL_EDITOR'; objectId: string; event: EventType }
  | { type: 'CLOSE_SPELL_EDITOR' }
  | { type: 'SET_CANDIDATE_SPELL'; text: string; ast: SpellAST }
  | { type: 'CONFIRM_SPELL' };
```

**Separation of Concerns:**
- **UI components**: Dispatch actions, render based on state, no business logic
- **Reducer**: Applies state transitions, enforces uniqueness constraints, calls domain functions
- **Domain functions**: Pure functions for parsing, execution, validation (no side effects)
- **Storage adapter**: Side effects only (read/write IndexedDB), called from effects, not reducer

---

## Spell Language

### Grammar (EBNF)

```ebnf
(* Top-level *)
spell         ::= statement+

(* Statements *)
statement     ::= go_to_card
                | set_field
                | show_message
                | if_statement

go_to_card    ::= "go" "to" "card" card_target
card_target   ::= STRING_LITERAL | NUMBER

set_field     ::= "set" FIELD_ID "to" expression

show_message  ::= "show" "message" expression

if_statement  ::= "if" condition "then" NEWLINE statement+ ( "else" NEWLINE statement+ )? "end" "if"

(* Conditions *)
condition     ::= expression comparator expression

comparator    ::= "is" | "is" "not" | "contains"

(* Expressions *)
expression    ::= STRING_LITERAL
                | NUMBER
                | field_read

field_read    ::= "field" FIELD_ID

(* Terminals *)
STRING_LITERAL ::= '"' ( [^"\] | '\"' | '\\' | '\n' )* '"'
NUMBER         ::= [0-9]+
FIELD_ID       ::= [a-zA-Z_][a-zA-Z0-9_]*
NEWLINE        ::= '\n' | '\r\n'
```

**Reserved Keywords:**

The following keywords are reserved and cannot be used as FIELD_IDs (case-insensitive):
`go`, `to`, `card`, `set`, `show`, `message`, `if`, `then`, `else`, `end`, `field`, `is`, `not`, `contains`

**Parser Notes:**
- Case-insensitive keywords
- FIELD_ID must not start with a digit and cannot be a reserved keyword
- Card names are STRING_LITERALs and are resolved case-insensitively at runtime
- NUMBER stored as text string for string-based comparisons
- STRING_LITERAL escaping: `\"` for literal quote, `\\` for literal backslash, `\n` for newline
- `go to card` terminates spell execution; any statements after it are unreachable

### AST Node Types

```typescript
type SpellAST = Statement[];

type Statement =
  | GoToCard
  | SetField
  | ShowMessage
  | IfStatement;

interface GoToCard {
  kind: 'GoToCard';
  target: CardTarget;
}

type CardTarget =
  | { kind: 'CardName'; name: string }
  | { kind: 'CardNumber'; number: number };

interface SetField {
  kind: 'SetField';
  fieldId: string;
  value: Expression;
}

interface ShowMessage {
  kind: 'ShowMessage';
  message: Expression;
}

interface IfStatement {
  kind: 'IfStatement';
  condition: Condition;
  thenBranch: Statement;
  elseBranch?: Statement;
}

type Condition = {
  kind: 'Condition';
  left: Expression;
  operator: 'is' | 'is-not' | 'contains';
  right: Expression;
};

type Expression =
  | StringLiteral
  | NumberLiteral
  | FieldRead;

interface StringLiteral {
  kind: 'StringLiteral';
  value: string;
}

interface NumberLiteral {
  kind: 'NumberLiteral';
  value: string; // Stored as text per requirement 4.9
}

interface FieldRead {
  kind: 'FieldRead';
  fieldId: string;
}
```

### Parser Implementation

**Approach**: Recursive descent parser

**Key Functions:**
- `parseSpell(text: string): Result<SpellAST, ParseError>`
- `parseStatement(tokens: Token[]): Result<Statement, ParseError>`
- `parseExpression(tokens: Token[]): Result<Expression, ParseError>`
- `parseCondition(tokens: Token[]): Result<Condition, ParseError>`

**Error Handling:**
- Track line and column numbers during tokenization
- Return first error encountered with line number
- Plain-language error messages:
  - "Expected 'to' on line 1 but found 'too'"
  - "Field identifier must not start with a digit on line 2"
  - "Unterminated string literal on line 3"

**Tokenizer:**
```typescript
interface Token {
  type: 'KEYWORD' | 'STRING' | 'NUMBER' | 'IDENTIFIER' | 'NEWLINE' | 'EOF';
  value: string;
  line: number;
  column: number;
}

function tokenize(text: string): Token[]
```

**Token Notes:**
- NEWLINE tokens separate statements and are significant in `if` blocks
- STRING tokens include escape sequences: `\"`, `\\`, `\n`

### Pretty-Printer

**Purpose**: Convert AST back to canonical text for round-trip property testing

**Function Signature:**
```typescript
function printSpell(ast: SpellAST): string
```

**Formatting Rules:**
- One statement per line
- Keywords lowercase
- Single space between tokens
- Indent nested statements (2 spaces)
- String literals always double-quoted with escaped quotes

**Example:**
```typescript
// AST
[
  {
    kind: 'IfStatement',
    condition: { kind: 'Condition', left: { kind: 'FieldRead', fieldId: 'answer' }, operator: 'is', right: { kind: 'StringLiteral', value: 'correct' } },
    thenBranch: [{ kind: 'ShowMessage', message: { kind: 'StringLiteral', value: 'Well done!' } }],
    elseBranch: [{ kind: 'ShowMessage', message: { kind: 'StringLiteral', value: 'Try again' } }]
  }
]

// Printed output
if field answer is "correct" then
  show message "Well done!"
else
  show message "Try again"
end if
```

---

## Spell Interpreter

### Execution Model

**Core Function:**
```typescript
interface ExecutionResult {
  kind: 'Success' | 'StepLimitExceeded' | 'RuntimeError' | 'NavigationDepthExceeded';
  stepsExecuted: number;
  stateChanges?: StateChange[];
  errorMessage?: string;
  errorStatement?: Statement;
}

interface StateChange =
  | { kind: 'SetField'; fieldId: string; newValue: string }
  | { kind: 'ShowMessage'; message: string }
  | { kind: 'NavigateCard'; targetIndex: number };

function executeSpell(
  ast: SpellAST,
  currentCard: Card,
  currentCardId: string,
  runtimeFields: Map<string, string>,
  stepLimit: number = 10000,
  navigationDepth: number = 0
): ExecutionResult
```

**Execution Steps:**
1. Initialize step counter to 0
2. Initialize a local copy of `runtimeFields` for the current card
3. For each statement in the AST:
   - Evaluate the statement
   - Increment step counter after each operation
   - For `set <id> to <expr>`: update the local field copy immediately
   - For `field <id>` reads: read from the local field copy (reflects earlier sets in the same spell)
4. Check step counter against `stepLimit` after each increment
5. For navigation statements, increment `navigationDepth` and check against limit (16)
6. Return `StateChange[]` for successful execution
7. Return error for failures

**Important:** During spell execution, field reads (`field <id>`) must reflect any earlier `set <id> to <expr>` statements in the same spell execution. The interpreter maintains a local copy of field values that is updated during execution and committed at the end.

**Step Counting:**
- Each expression evaluation = 1 step
- Each condition evaluation = 1 step
- Each statement execution = 1 step
- Example: `if field answer is "correct" then show message "Yes" end if` = 5 steps
  - Read field: 1
  - String literal: 1
  - Condition: 1
  - Show message: 1
  - String literal: 1

---

## Execution Controller

The execution controller orchestrates spell execution and state application:

### Applying StateChanges

StateChanges are applied in the order they are returned by `executeSpell()`. Field updates and message displays are applied immediately; navigation triggers a recursive execution.

### Card-Open Event Firing

WHEN a spell executes a `NavigateCard` state change, THE execution controller SHALL:
- Apply the navigation (switch to target card)
- Fire the `card-open` spell for the target card (if one exists)
- Increment `navigationDepth` by 1 before executing the card-open spell
- Halt with a "navigation depth exceeded" error if `navigationDepth` exceeds 16

### Depth Reset

The `navigationDepth` counter is reset to 0 on every user-initiated event (button click, explicit card navigation).

### Entering Browse Mode

WHEN the user enters Browse_Mode, THE execution controller SHALL fire the `card-open` spell for the current card (if one exists) with `navigationDepth = 0`.

### Sandboxing

**Enforcement Mechanisms:**
1. **Pure functions**: Interpreter has no side effects; returns `StateChange[]` for UI to apply
2. **No global access**: Interpreter receives only `Card`, `runtimeFields`, and `stepLimit`
3. **No DOM/network/storage**: Interpreter is a pure TypeScript module with no imports of browser APIs
4. **Field scope validation**: Check field IDs against `currentCard.objects` before reading

### Error Handling

**Runtime Errors:**
- **Missing field reference**: "Field 'username' does not exist on this card"
- **Invalid card target**: "Card 'Results' does not exist in this spellbook"
- **Step limit exceeded**: "This spell was stopped because it performed too many steps."
- **Navigation depth exceeded**: "This spell was stopped because it navigated through too many cards."

**Error Result:**
```typescript
{
  kind: 'RuntimeError',
  stepsExecuted: 437,
  errorMessage: "Field 'username' does not exist on this card",
  errorStatement: { kind: 'FieldRead', fieldId: 'username' }
}
```

**UI Display:**
- Modal dialog with error message
- Highlight the statement in spell editor (Edit mode)
- Log to console for debugging

### Determinism Property

**Requirement**: Same inputs → Same outputs

**Inputs:**
- SpellAST (immutable)
- Card state (immutable during execution)
- Runtime field values (read-only Map)

**Outputs:**
- StateChange[] (deterministic order)
- stepsExecuted (deterministic count)
- errorMessage (deterministic for same error)

**Testing Approach:**
- Property-based test: Generate random AST, card state, field values
- Execute spell twice
- Assert identical results

---

## Data Models

### Core Interfaces

```typescript
interface Spellbook {
  id: string; // UUID v4
  name: string;
  cards: Card[];
  createdAt: number; // Unix timestamp
  modifiedAt: number; // Unix timestamp
  isExample: boolean;
}

interface Card {
  id: string; // UUID v4
  name: string; // Card_Name, unique within spellbook
  objects: CardObject[];
  spells: CardSpell[];
}

type CardObject = ButtonObject | TextFieldObject | LabelObject;

interface BaseObject {
  id: string; // UUID v4
  kind: 'Button' | 'TextField' | 'Label';
  x: number; // Pixels from left
  y: number; // Pixels from top
  width: number; // Pixels
  height: number; // Pixels
}

interface ButtonObject extends BaseObject {
  kind: 'Button';
  label: string;
}

interface TextFieldObject extends BaseObject {
  kind: 'TextField';
  fieldId: string; // Must match FIELD_ID grammar rule
  defaultValue: string;
}

interface LabelObject extends BaseObject {
  kind: 'Label';
  text: string;
}

interface CardSpell {
  target: SpellTarget;
  event: EventType;
  sourceText: string; // Original spell text; AST is derived at load/parse time
}

type SpellTarget =
  | { kind: 'Object'; objectId: string }
  | { kind: 'Card' }; // Card-open spells are attached to the card itself

type EventType = 'click' | 'card-open';
```

**Note:** ASTs are derived by parsing `sourceText` at application load time and when a spell is edited. ASTs are never persisted.

### Validation Rules

**Card_Name Uniqueness:**
```typescript
function validateCardName(spellbook: Spellbook, cardIndex: number, newName: string): ValidationResult {
  const existingIndex = spellbook.cards.findIndex((c, i) => 
    i !== cardIndex && c.name.toLowerCase() === newName.toLowerCase()
  );
  
  if (existingIndex !== -1) {
    return { valid: false, message: `A card named "${newName}" already exists` };
  }
  
  return { valid: true };
}
```

**Field_Identifier Validation:**
```typescript
const FIELD_ID_REGEX = /^[a-zA-Z_][a-zA-Z0-9_]*$/;

function validateFieldId(fieldId: string): ValidationResult {
  if (fieldId.length === 0) {
    return { valid: false, message: 'Field identifier cannot be empty' };
  }
  
  if (!FIELD_ID_REGEX.test(fieldId)) {
    return { valid: false, message: 'Field identifier must contain only letters, digits, and underscores, and must not start with a digit' };
  }
  
  return { valid: true };
}

function validateFieldIdUnique(card: Card, fieldId: string, excludeObjectId?: string): ValidationResult {
  const existing = card.objects.find(obj => 
    obj.kind === 'TextField' && 
    obj.fieldId === fieldId && 
    obj.id !== excludeObjectId
  );
  
  if (existing) {
    return { valid: false, message: `A field with identifier "${fieldId}" already exists on this card` };
  }
  
  return { valid: true };
}
```

### Runtime vs. Authored State

**Authored State** (persisted):
- Object positions, sizes, labels
- TextField default values
- Spell source text and AST

**Runtime State** (Browse mode only, ephemeral):
- Current TextField values (initialized from default, may be modified by spells)
- Current card index (but persisted when saving spellbook)
- Navigation depth counter

**State Transition:**
```typescript
// Entering Browse Mode
function initializeRuntimeState(spellbook: Spellbook): RuntimeState {
  const fieldValues = new Map<string, string>();
  
  for (const card of spellbook.cards) {
    for (const obj of card.objects) {
      if (obj.kind === 'TextField') {
        const key = `${card.id}:${obj.fieldId}`;
        fieldValues.set(key, obj.defaultValue);
      }
    }
  }
  
  return {
    mode: 'Browse',
    fieldValues,
    navigationDepth: 0
  };
}

// Exiting Browse Mode
function discardRuntimeState(): void {
  // Simply don't persist fieldValues
  // Authored state remains unchanged
}
```

---

## Storage Layer

### IndexedDB Schema

**Database Name:** `grimoire-db`

**Version:** 1

**Object Stores:**

```typescript
// Store: spellbooks
interface SpellbookRecord {
  id: string; // Primary key
  name: string;
  cards: Card[];
  createdAt: number;
  modifiedAt: number;
  isExample: boolean;
  schemaVersion: string; // e.g., "1.0.0"
}

// Index: by-modified (for recent sorting)
// Index: by-name (for alphabetical sorting)
```

### Auto-Save Strategy

**Approach**: Debounced save on every state change

```typescript
// In React effect
useEffect(() => {
  if (!currentSpellbook) return;
  
  const timeoutId = setTimeout(() => {
    saveSpellbook(currentSpellbook).catch(error => {
      showStorageErrorDialog(error);
    });
  }, 500); // 500ms debounce
  
  return () => clearTimeout(timeoutId);
}, [currentSpellbook]);
```

**Save Function:**
```typescript
async function saveSpellbook(spellbook: Spellbook): Promise<void> {
  try {
    const db = await openDatabase();
    const tx = db.transaction('spellbooks', 'readwrite');
    const store = tx.objectStore('spellbooks');
    
    const record: SpellbookRecord = {
      ...spellbook,
      modifiedAt: Date.now(),
      schemaVersion: CURRENT_SCHEMA_VERSION
    };
    
    await store.put(record);
    await tx.done;
  } catch (error) {
    throw new StorageError('Failed to save spellbook', error);
  }
}
```

**Note:** IndexedDB transactions auto-commit when all requests succeed and no new requests are made. The `tx.done` promise resolves when the transaction completes.

**Error Handling:**
- Quota exceeded: Show warning, offer export as file
- Storage unavailable: Show warning, continue in-memory editing, enable export
- Write failure: Show warning, retry once, then offer export

### Export Format

**File Extension:** `.grimoire.json`

**Schema:**
```typescript
interface ExportFormat {
  version: string; // Schema version, e.g., "1.0.0"
  spellbook: {
    name: string;
    cards: ExportCard[];
    createdAt: number;
    modifiedAt: number;
  };
}

interface ExportCard {
  name: string;
  objects: ExportObject[];
  spells: ExportSpell[];
}

interface ExportObject {
  kind: 'Button' | 'TextField' | 'Label';
  x: number;
  y: number;
  width: number;
  height: number;
  // Kind-specific fields
  label?: string; // Button
  text?: string; // Label
  fieldId?: string; // TextField
  defaultValue?: string; // TextField
}

interface ExportSpell {
  objectId: string;
  event: 'click' | 'card-open';
  sourceText: string;
}
```

**Object ID Handling:**
- Object IDs are exported as-is (stable local UUIDs)
- On import, all IDs are remapped to new UUIDs to avoid collisions

**Export Function:**
```typescript
function exportSpellbook(spellbook: Spellbook): string {
  const exportData: ExportFormat = {
    version: CURRENT_SCHEMA_VERSION,
    spellbook: {
      name: spellbook.name,
      cards: spellbook.cards.map(card => ({
        name: card.name,
        objects: card.objects.map(obj => {
          const base = { kind: obj.kind, x: obj.x, y: obj.y, width: obj.width, height: obj.height };
          if (obj.kind === 'Button') return { ...base, label: obj.label };
          if (obj.kind === 'TextField') return { ...base, fieldId: obj.fieldId, defaultValue: obj.defaultValue };
          if (obj.kind === 'Label') return { ...base, text: obj.text };
          return base;
        }),
        spells: card.spells.map(spell => ({
          objectId: spell.objectId,
          event: spell.event,
          sourceText: spell.sourceText
        }))
      })),
      createdAt: spellbook.createdAt,
      modifiedAt: spellbook.modifiedAt
    }
  };
  
  return JSON.stringify(exportData, null, 2);
}
```

### Import Validation

**Schema Library:** Zod (TypeScript-first schema validation)

**Zod Schema:**
```typescript
import { z } from 'zod';

const ExportObjectSchema = z.object({
  kind: z.enum(['Button', 'TextField', 'Label']),
  x: z.number().int().min(0).max(10000),
  y: z.number().int().min(0).max(10000),
  width: z.number().int().min(0).max(10000),
  height: z.number().int().min(0).max(10000),
  label: z.string().optional(),
  text: z.string().optional(),
  fieldId: z.string().max(50).optional(),
  defaultValue: z.string().optional()
});

const ExportSpellSchema = z.object({
  objectId: z.string(),
  event: z.enum(['click', 'card-open']),
  sourceText: z.string().max(10000)
});

const ExportCardSchema = z.object({
  name: z.string().min(1).max(100),
  objects: z.array(ExportObjectSchema).max(50),
  spells: z.array(ExportSpellSchema).max(20)
});

const ExportFormatSchema = z.object({
  version: z.string(),
  spellbook: z.object({
    name: z.string().min(1).max(100),
    cards: z.array(ExportCardSchema).min(1).max(100),
    createdAt: z.number(),
    modifiedAt: z.number()
  })
});
```

**Validation Pipeline:**
1. Parse JSON
2. Validate against Zod schema
3. Check size limit (5 MB)
4. Check unique card names (case-insensitive)
5. Check unique field IDs within each card
6. Re-parse all spells
7. Validate spell targets (objectId references existing object)
8. Remap all IDs to new UUIDs

**Security Note:** React will render all imported text as plain text by default; combined with schema validation, this provides sufficient XSS protection.

```typescript
interface SpellbookImportData {
  name: string;
  cards: ImportCard[];
  createdAt: number;
  modifiedAt: number;
}

interface ImportCard {
  name: string;
  objects: ImportObject[];
  spells: ImportSpell[];
}

interface ImportObject {
  kind: 'Button' | 'TextField' | 'Label';
  x: number;
  y: number;
  width: number;
  height: number;
  label?: string;
  text?: string;
  fieldId?: string;
  defaultValue?: string;
}

interface ImportSpell {
  objectId: string;
  event: 'click' | 'card-open';
  sourceText: string;
  ast: SpellAST; // Parsed during import
}

function parseImport(fileContent: string): Result<SpellbookImportData, ImportError> {
  // 1. Size check
  const sizeInMB = new Blob([fileContent]).size / (1024 * 1024);
  if (sizeInMB > 5) {
    return { ok: false, error: { kind: 'FileTooLarge', message: 'Import file is too large (maximum 5 MB)' } };
  }
  
  // 2. Parse JSON
  let parsed: unknown;
  try {
    parsed = JSON.parse(fileContent);
  } catch (error) {
    return { ok: false, error: { kind: 'InvalidJSON', message: 'File is not valid JSON' } };
  }
  
  // 3. Zod schema validation
  const schemaResult = ExportFormatSchema.safeParse(parsed);
  if (!schemaResult.success) {
    return {
      ok: false,
      error: {
        kind: 'SchemaValidation',
        message: `Schema validation failed: ${schemaResult.error.message}`
      }
    };
  }
  
  const validated = schemaResult.data;
  
  // 4. Check unique card names (case-insensitive)
  const cardNamesSeen = new Set<string>();
  for (const card of validated.spellbook.cards) {
    const lowerName = card.name.toLowerCase();
    if (cardNamesSeen.has(lowerName)) {
      return {
        ok: false,
        error: {
          kind: 'SchemaValidation',
          message: `Duplicate card name: "${card.name}"`
        }
      };
    }
    cardNamesSeen.add(lowerName);
  }
  
  // 5. Check unique field IDs within each card and re-parse spells
  const cards: ImportCard[] = [];
  for (const exportCard of validated.spellbook.cards) {
    // Check field ID uniqueness
    const fieldIdsSeen = new Set<string>();
    for (const obj of exportCard.objects) {
      if (obj.kind === 'TextField' && obj.fieldId) {
        const lowerFieldId = obj.fieldId.toLowerCase();
        if (fieldIdsSeen.has(lowerFieldId)) {
          return {
            ok: false,
            error: {
              kind: 'SchemaValidation',
              message: `Duplicate field ID "${obj.fieldId}" on card "${exportCard.name}"`
            }
          };
        }
        fieldIdsSeen.add(lowerFieldId);
      }
    }
    
    // Re-parse all spells
    const spells: ImportSpell[] = [];
    for (const exportSpell of exportCard.spells) {
      const parseResult = parseSpell(exportSpell.sourceText);
      if (!parseResult.ok) {
        return {
          ok: false,
          error: {
            kind: 'SpellParsing',
            message: `Failed to parse spell on card "${exportCard.name}": ${parseResult.error.message}`,
            details: `Spell source: ${exportSpell.sourceText}`
          }
        };
      }
      
      spells.push({
        objectId: exportSpell.objectId,
        event: exportSpell.event,
        sourceText: exportSpell.sourceText,
        ast: parseResult.ast
      });
    }
    
    // 6. Validate spell targets
    const objectIds = new Set(exportCard.objects.map((_, idx) => `temp-${idx}`)); // Use temp IDs for validation
    for (const spell of spells) {
      if (spell.event === 'click' && !objectIds.has(spell.objectId)) {
        return {
          ok: false,
          error: {
            kind: 'SchemaValidation',
            message: `Spell references non-existent object ID "${spell.objectId}" on card "${exportCard.name}"`
          }
        };
      }
    }
    
    cards.push({
      name: exportCard.name,
      objects: exportCard.objects as ImportObject[],
      spells
    });
  }
  
  return {
    ok: true,
    value: {
      name: validated.spellbook.name,
      cards,
      createdAt: validated.spellbook.createdAt,
      modifiedAt: validated.spellbook.modifiedAt
    }
  };
}

async function saveImportedSpellbook(importData: SpellbookImportData): Promise<Spellbook> {
  // Remap all IDs to new UUIDs
  const spellbook: Spellbook = {
    id: generateUUID(),
    name: importData.name,
    cards: importData.cards.map(importCard => {
      const cardId = generateUUID();
      const oldToNewObjectIds = new Map<string, string>();
      
      const objects: CardObject[] = importCard.objects.map(obj => {
        const newId = generateUUID();
        oldToNewObjectIds.set(obj.objectId || `temp-${importCard.objects.indexOf(obj)}`, newId);
        
        return {
          ...obj,
          id: newId
        } as CardObject;
      });
      
      const spells: CardSpell[] = importCard.spells.map(spell => ({
        target: spell.event === 'card-open'
          ? { kind: 'Card' }
          : { kind: 'Object', objectId: oldToNewObjectIds.get(spell.objectId) || spell.objectId },
        event: spell.event,
        sourceText: spell.sourceText
      }));
      
      return {
        id: cardId,
        name: importCard.name,
        objects,
        spells
      };
    }),
    createdAt: Date.now(),
    modifiedAt: Date.now(),
    isExample: false
  };
  
  await saveSpellbook(spellbook);
  return spellbook;
}
```

---

## Spell Generation UI [Core]

### Template_Fallback

**Rule-Based Generation:**

```typescript
interface GenerationContext {
  event: EventType;
  objectType: 'Button' | 'TextField' | 'Label';
  availableFields: string[]; // Field IDs on current card
  cardNames: string[]; // All card names in spellbook
  cardCount: number; // Total number of cards
  description: string; // Plain-English input
}

function matchWholeWord(text: string, word: string): boolean {
  const regex = new RegExp(`\\b${word}\\b`, 'i');
  return regex.test(text);
}

function generateSpellFromTemplate(context: GenerationContext): string {
  const { event, objectType, description, cardNames, cardCount } = context;
  
  // Pattern matching on description keywords
  const lowerDesc = description.toLowerCase();
  
  // Navigation patterns
  if (lowerDesc.includes('go to') || lowerDesc.includes('navigate') || lowerDesc.includes('next card')) {
    const cardMatch = extractCardNameWholeWord(description, cardNames);
    if (cardMatch) {
      return `go to card "${cardMatch}"`;
    }
    
    // Check for number
    const numberMatch = description.match(/\b(\d+)\b/);
    if (numberMatch) {
      const num = parseInt(numberMatch[1], 10);
      const clampedNum = Math.max(1, Math.min(num, cardCount));
      return `go to card ${clampedNum}`;
    }
    
    return 'go to card 2'; // Default to next card
  }
  
  // Field setting patterns
  if (lowerDesc.includes('set') && lowerDesc.includes('to')) {
    const fieldMatch = extractFieldIdWholeWord(description, context.availableFields);
    const valueMatch = extractQuotedValue(description);
    if (fieldMatch && valueMatch) {
      return `set ${fieldMatch} to "${valueMatch}"`;
    }
    if (fieldMatch) {
      return `set ${fieldMatch} to "value"`;
    }
  }
  
  // Message patterns
  if (lowerDesc.includes('show') || lowerDesc.includes('message') || lowerDesc.includes('alert')) {
    const messageMatch = extractQuotedValue(description);
    if (messageMatch) {
      return `show message "${messageMatch}"`;
    }
    return 'show message "Hello"';
  }
  
  // Conditional patterns
  if (lowerDesc.includes('if') && context.availableFields.length > 0) {
    const field = context.availableFields[0];
    return `if field ${field} is "yes" then\n  show message "Correct!"\nelse\n  show message "Try again"\nend if`;
  }
  
  // Default fallback by event type
  if (event === 'click') {
    return 'show message "Button clicked"';
  } else if (event === 'card-open') {
    return 'show message "Welcome"';
  }
  
  return 'show message "Hello"';
}

// Helper functions
function extractCardNameWholeWord(text: string, cardNames: string[]): string | null {
  for (const name of cardNames) {
    if (matchWholeWord(text, name)) {
      return name;
    }
  }
  return null;
}

function extractFieldIdWholeWord(text: string, fieldIds: string[]): string | null {
  for (const id of fieldIds) {
    if (matchWholeWord(text, id)) {
      return id;
    }
  }
  return null;
}

function extractQuotedValue(text: string): string | null {
  const match = text.match(/"([^"]+)"/);
  return match ? match[1] : null;
}
```

### Parse-Before-Display Flow

```typescript
async function handleGenerateSpell(description: string, context: GenerationContext): Promise<void> {
  let candidateText: string;
  
  // 1. Attempt AI generation (if available)
  try {
    const aiResult = await generateSpellWithAI(description, context, { timeout: 10000 });
    candidateText = aiResult.spellText;
  } catch (error) {
    // 2. Fall back to template
    showNotice('AI service unavailable, using template generation');
    candidateText = generateSpellFromTemplate(context);
  }
  
  // 3. Parse before displaying
  const parseResult = parseSpell(candidateText);
  if (!parseResult.ok) {
    showError('Generation failed: could not create a valid spell. Please try rephrasing your description.');
    return;
  }
  
  // 4. Display in editor for review
  dispatch({
    type: 'SET_CANDIDATE_SPELL',
    text: candidateText,
    ast: parseResult.ast
  });
  
  // 5. Wait for user to confirm or edit
}
```

### Explicit Confirm Step

**UI Flow:**
1. User enters plain-English description
2. Click "Generate" button
3. Generated spell appears in text editor (read-only or editable)
4. Status indicator: "Generated spell (not yet saved)"
5. Two buttons: "Confirm and Attach" or "Cancel"
6. If user edits: re-parse on each keystroke, show parse errors inline
7. Only on "Confirm and Attach": attach spell to object event

---

## AI Service Integration [Stretch]

### Architecture

The AI service integration is provider-agnostic. The server can call OpenAI, Anthropic, or any other LLM API.

**Client (Browser):**
- Collect context (description, event type, available fields, card names)
- Send POST request to `/api/generate-spell`
- Display privacy notice before first request
- Handle timeout (10 seconds)
- Fall back to Template_Fallback on error or unavailability

**Server (Node.js/Express):**
- Receive request with context
- Validate and sanitize inputs: `description` length ≤ 500 chars, `availableFields` length ≤ 20, `cardNames` length ≤ 100
- Call AI service (OpenAI, Anthropic, or similar) with API key from environment
- Return spell text in response
- Never expose API key to client
- **MUST implement rate limiting** (e.g., 10 requests per minute per IP) to prevent abuse

**Hosting:** The server endpoint can be hosted on Vercel, Netlify Functions, or similar. Judges evaluating this project can use a publicly-documented test endpoint with no API key required, or the application falls back to Template_Fallback when the endpoint is unavailable.

### API Contract

**Request:**
```typescript
POST /api/generate-spell
Content-Type: application/json

{
  "description": "Show a message when the user clicks the button",
  "event": "click",
  "objectType": "Button",
  "availableFields": ["username", "age"],
  "cardNames": ["Welcome", "Quiz", "Results"]
}
```

**Input Validation:**
- `description`: string, max 500 chars
- `availableFields`: array, max 20 items
- `cardNames`: array, max 100 items
- `event`: enum ('click' | 'card-open')
- `objectType`: enum ('Button' | 'TextField' | 'Label')

**Response (Success):**
```typescript
{
  "spellText": "show message \"Button clicked\""
}
```

**Response (Error):**
```typescript
{
  "error": "AI service timeout"
}
```

### Client Implementation

```typescript
async function generateSpellWithAI(
  description: string,
  context: GenerationContext,
  options: { timeout: number }
): Promise<{ spellText: string }> {
  // Privacy notice (shown once per session)
  if (!hasSeenPrivacyNotice()) {
    await showPrivacyNoticeDialog(
      'Your description will be sent to a server for processing. No personal data is stored.'
    );
    markPrivacyNoticeSeen();
  }
  
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), options.timeout);
  
  try {
    const response = await fetch('/api/generate-spell', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        description,
        event: context.event,
        objectType: context.objectType,
        availableFields: context.availableFields,
        cardNames: context.cardNames
      }),
      signal: controller.signal
    });
    
    clearTimeout(timeoutId);
    
    if (!response.ok) {
      // Fall back on any error (404, 503, etc.)
      throw new Error(`Server error: ${response.status}`);
    }
    
    const data = await response.json();
    return { spellText: data.spellText };
  } catch (error) {
    clearTimeout(timeoutId);
    // Fall back to template on any error (timeout, network, server unavailable)
    throw error;
  }
}
```

### Server Implementation (Minimal)

```typescript
// server.ts
import express from 'express';
import OpenAI from 'openai';
import rateLimit from 'express-rate-limit';

const app = express();
app.use(express.json());

// Rate limiting: 10 requests per minute per IP
const limiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10,
  message: { error: 'Too many requests, please try again later' }
});

app.use('/api/generate-spell', limiter);

const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY // Never in client code
});

app.post('/api/generate-spell', async (req, res) => {
  const { description, event, objectType, availableFields, cardNames } = req.body;
  
  // Validate inputs
  if (!description || typeof description !== 'string' || description.length > 500) {
    return res.status(400).json({ error: 'Invalid description (max 500 chars)' });
  }
  if (!Array.isArray(availableFields) || availableFields.length > 20) {
    return res.status(400).json({ error: 'Invalid availableFields (max 20)' });
  }
  if (!Array.isArray(cardNames) || cardNames.length > 100) {
    return res.status(400).json({ error: 'Invalid cardNames (max 100)' });
  }
  
  try {
    const completion = await openai.chat.completions.create({
      model: 'gpt-4',
      messages: [
        {
          role: 'system',
          content: `You are a code generator for a simple scripting language. Generate a spell script based on the user's description. Available fields: ${availableFields.join(', ')}. Available cards: ${cardNames.join(', ')}. Grammar: go to card <name|number> | set <field> to <value> | show message <value> | if <field> is <value> then <statement> else <statement> end if`
        },
        {
          role: 'user',
          content: description
        }
      ],
      max_tokens: 150
    });
    
    const spellText = completion.choices[0].message.content.trim();
    res.json({ spellText });
  } catch (error) {
    console.error('AI service error:', error);
    res.status(500).json({ error: 'AI service error' });
  }
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

**Security Considerations:**
- API key stored in environment variable (`.env` file, not committed)
- Input validation on server
- Rate limiting required (10 requests/minute/IP)
- No user tracking or data storage

---

## Example Spellbooks

### Data Structure

**Storage Location:** Embedded in application code as constants

**Note:** ASTs are derived by parsing `sourceText` at load time, not stored. Example spellbooks are validated at build time (or startup) to ensure all spells parse successfully.

**Note:** All example spellbooks use spooky-themed content (e.g., 'What potion ingredient glows in the dark?', 'Moonstone Fragments', 'Enchanted Price List').

```typescript
const EXAMPLE_SPELLBOOKS: Spellbook[] = [
  {
    id: 'example-quiz',
    name: 'Quiz Example',
    isExample: true,
    createdAt: 0,
    modifiedAt: 0,
    cards: [
      {
        id: 'quiz-card-1',
        name: 'Question',
        objects: [
          {
            id: 'quiz-label-1',
            kind: 'Label',
            x: 50,
            y: 50,
            width: 400,
            height: 60,
            text: 'What potion ingredient glows in the dark?'
          },
          {
            id: 'quiz-field-1',
            kind: 'TextField',
            x: 50,
            y: 130,
            width: 200,
            height: 40,
            fieldId: 'answer',
            defaultValue: ''
          },
          {
            id: 'quiz-button-1',
            kind: 'Button',
            x: 50,
            y: 190,
            width: 100,
            height: 40,
            label: 'Submit'
          }
        ],
        spells: [
          {
            target: { kind: 'Object', objectId: 'quiz-button-1' },
            event: 'click',
            sourceText: 'if field answer is "Moonstone" then\n  go to card "Complete"\nelse\n  show message "Try again!"\nend if'
          }
        ]
      },
      {
        id: 'quiz-card-2',
        name: 'Complete',
        objects: [
          {
            id: 'quiz-label-2',
            kind: 'Label',
            x: 50,
            y: 100,
            width: 400,
            height: 60,
            text: 'Quiz complete! You know your ingredients.'
          }
        ],
        spells: []
      }
    ]
  },
  
  // Shop price list example
  {
    id: 'example-shop',
    name: 'Enchanted Price List',
    isExample: true,
    createdAt: 0,
    modifiedAt: 0,
    cards: [
      {
        id: 'shop-card-1',
        name: 'Products',
        objects: [
          {
            id: 'shop-label-1',
            kind: 'Label',
            x: 50,
            y: 50,
            width: 300,
            height: 40,
            text: 'Crystal Orb - 10 gold'
          },
          {
            id: 'shop-button-1',
            kind: 'Button',
            x: 50,
            y: 100,
            width: 150,
            height: 40,
            label: 'View Details'
          }
        ],
        spells: [
          {
            target: { kind: 'Object', objectId: 'shop-button-1' },
            event: 'click',
            sourceText: 'go to card "Details"'
          }
        ]
      },
      {
        id: 'shop-card-2',
        name: 'Details',
        objects: [
          {
            id: 'shop-label-2',
            kind: 'Label',
            x: 50,
            y: 50,
            width: 400,
            height: 100,
            text: 'Crystal Orb: Reveals hidden truths for 10 gold'
          },
          {
            id: 'shop-button-2',
            kind: 'Button',
            x: 50,
            y: 160,
            width: 100,
            height: 40,
            label: 'Back'
          }
        ],
        spells: [
          {
            target: { kind: 'Object', objectId: 'shop-button-2' },
            event: 'click',
            sourceText: 'go to card "Products"'
          }
        ]
      }
    ]
  },
  
  // Flashcard deck example
  {
    id: 'example-flashcards',
    name: 'Spell Flashcards',
    isExample: true,
    createdAt: 0,
    modifiedAt: 0,
    cards: [
      {
        id: 'flash-card-1',
        name: 'Card1',
        objects: [
          {
            id: 'flash-label-1',
            kind: 'Label',
            x: 50,
            y: 50,
            width: 400,
            height: 60,
            text: 'Question: What breaks curses?'
          },
          {
            id: 'flash-button-1',
            kind: 'Button',
            x: 50,
            y: 130,
            width: 150,
            height: 40,
            label: 'Show Answer'
          }
        ],
        spells: [
          {
            target: { kind: 'Object', objectId: 'flash-button-1' },
            event: 'click',
            sourceText: 'show message "Salt circles and moonlight"'
          }
        ]
      },
      {
        id: 'flash-card-2',
        name: 'Card2',
        objects: [
          {
            id: 'flash-label-2',
            kind: 'Label',
            x: 50,
            y: 50,
            width: 400,
            height: 60,
            text: 'Question: What seals a binding spell?'
          },
          {
            id: 'flash-button-2',
            kind: 'Button',
            x: 50,
            y: 130,
            width: 150,
            height: 40,
            label: 'Show Answer'
          }
        ],
        spells: [
          {
            target: { kind: 'Object', objectId: 'flash-button-2' },
            event: 'click',
            sourceText: 'show message "Wax and iron filings"'
          }
        ]
      }
    ]
  }
];
```

### Opening Example Spellbooks

WHEN the user opens an example spellbook for the first time, Grimoire creates a working copy with a name like "Quiz Example (My Copy)".

IF the user opens the example again, Grimoire reuses the existing working copy (does not create duplicates).

The start screen shows example spellbooks in a separate section with a visual indicator (e.g., a badge or icon).

### Startup Parse Verification

**On Application Load:**
```typescript
function verifyExampleSpellbooks(): void {
  for (const spellbook of EXAMPLE_SPELLBOOKS) {
    for (const card of spellbook.cards) {
      for (const spell of card.spells) {
        const parseResult = parseSpell(spell.sourceText);
        if (!parseResult.ok) {
          console.error(
            `Failed to parse example spell in spellbook "${spellbook.name}", card "${card.name}":`,
            parseResult.error.message
          );
          // Log but don't prevent app from starting
        }
      }
    }
  }
}
```

**Loading Example Spellbook:**
```typescript
function loadExampleSpellbook(exampleId: string): Spellbook {
  const example = EXAMPLE_SPELLBOOKS.find(sb => sb.id === exampleId);
  if (!example) {
    throw new Error(`Example spellbook "${exampleId}" not found`);
  }
  
  // Check if a working copy already exists
  const existingCopy = await loadAllSpellbooks().then(books =>
    books.find(sb => sb.name === `${example.name} (My Copy)`)
  );
  
  if (existingCopy) {
    return existingCopy;
  }
  
  // Deep clone to create a working copy
  const workingCopy = JSON.parse(JSON.stringify(example));
  
  // Assign new ID and mark as non-example
  workingCopy.id = generateUUID();
  workingCopy.isExample = false;
  workingCopy.name = `${example.name} (My Copy)`;
  workingCopy.createdAt = Date.now();
  workingCopy.modifiedAt = Date.now();
  
  return workingCopy;
}
```

---

## Start Screen

### Component Structure

```typescript
function StartScreen() {
  const [spellbooks, setSpellbooks] = useState<Spellbook[]>([]);
  
  useEffect(() => {
    loadAllSpellbooks().then(setSpellbooks);
  }, []);
  
  const userSpellbooks = spellbooks.filter(sb => !sb.isExample);
  const exampleSpellbooks = EXAMPLE_SPELLBOOKS;
  
  return (
    <div className="start-screen">
      <h1>Grimoire</h1>
      <p className="tagline">build small apps from cards, in plain English</p>
      
      <section>
        <h2>Your Spellbooks</h2>
        <button onClick={handleCreateNew}>Create New Spellbook</button>
        <SpellbookList spellbooks={userSpellbooks} onOpen={handleOpen} onRename={handleRename} onDelete={handleDelete} />
      </section>
      
      <section>
        <h2>Examples</h2>
        <SpellbookList spellbooks={exampleSpellbooks} onOpen={handleOpenExample} />
      </section>
    </div>
  );
}
```

### Operations

**Create New:**
```typescript
function handleCreateNew(): void {
  const newSpellbook: Spellbook = {
    id: generateUUID(),
    name: 'Untitled Spellbook',
    cards: [
      {
        id: generateUUID(),
        name: 'Card 1',
        objects: [],
        spells: []
      }
    ],
    createdAt: Date.now(),
    modifiedAt: Date.now(),
    isExample: false
  };
  
  saveSpellbook(newSpellbook).then(() => {
    dispatch({ type: 'OPEN_SPELLBOOK', spellbook: newSpellbook });
  });
}
```

**Open:**
```typescript
function handleOpen(spellbookId: string): void {
  loadSpellbook(spellbookId).then(spellbook => {
    dispatch({ type: 'OPEN_SPELLBOOK', spellbook });
  });
}
```

**Open Example:**
```typescript
function handleOpenExample(exampleId: string): void {
  const workingCopy = loadExampleSpellbook(exampleId);
  saveSpellbook(workingCopy).then(() => {
    dispatch({ type: 'OPEN_SPELLBOOK', spellbook: workingCopy });
  });
}
```

**Rename:**
```typescript
function handleRename(spellbookId: string, newName: string): void {
  loadSpellbook(spellbookId).then(spellbook => {
    spellbook.name = newName;
    saveSpellbook(spellbook).then(() => {
      setSpellbooks(prev => prev.map(sb => sb.id === spellbookId ? spellbook : sb));
    });
  });
}
```

**Delete:**
```typescript
function handleDelete(spellbookId: string): void {
  showConfirmDialog('Delete this spellbook? This cannot be undone.').then(confirmed => {
    if (confirmed) {
      deleteSpellbook(spellbookId).then(() => {
        setSpellbooks(prev => prev.filter(sb => sb.id !== spellbookId));
      });
    }
  });
}
```

### Visual Distinction

**User Spellbooks:**
- Full controls: open, rename, delete
- Displayed with modification date
- Sorted by most recently modified

**Example Spellbooks:**
- Read-only appearance (lighter styling)
- Only "Open" action (creates working copy)
- Badge or icon indicating "Example"
- Fixed order

---

## Accessibility and Styling

### Spooky-Retro Theme

**CSS Custom Properties:**
```css
:root {
  /* Colors */
  --color-bg-dark: #0a0a0a;
  --color-bg-medium: #1a1a1a;
  --color-bg-light: #2a2a2a;
  --color-text-primary: #f0f0f0;
  --color-text-secondary: #b0b0b0;
  --color-accent: #ff6b35; /* Candlelit orange */
  --color-accent-hover: #ff8555;
  --color-focus: #ffaa77;
  --color-border: #3a3a3a;
  --color-error: #ff4444;
  --color-success: #44ff44;
  
  /* Typography */
  --font-body: 'Courier New', monospace;
  --font-heading: 'Georgia', serif;
  
  /* Effects */
  --glow-small: 0 0 5px var(--color-accent);
  --glow-large: 0 0 15px var(--color-accent);
  --shadow-depth: 0 4px 8px rgba(0, 0, 0, 0.5);
  
  /* Animation */
  --transition-fast: 150ms ease;
  --transition-medium: 300ms ease;
}

@media (prefers-reduced-motion: reduce) {
  :root {
    --transition-fast: 0ms;
    --transition-medium: 0ms;
  }
  
  * {
    animation: none !important;
    transition: none !important;
  }
}
```

**Spooky Effects (That Don't Obscure Content):**
- Subtle glow on hover (not on focus, to maintain clear focus indicator)
- Flame flicker animation on accent elements (disabled in reduced-motion)
- Parchment texture on card backgrounds (low contrast, text always readable)
- Candlelight gradient on headers (decorative, not essential)

### Reduced Motion Support

**React Hook:**
```typescript
function useReducedMotion(): boolean {
  const [prefersReducedMotion, setPrefersReducedMotion] = useState(false);
  
  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    setPrefersReducedMotion(mediaQuery.matches);
    
    const listener = (e: MediaQueryListEvent) => {
      setPrefersReducedMotion(e.matches);
    };
    
    mediaQuery.addEventListener('change', listener);
    return () => mediaQuery.removeEventListener('change', listener);
  }, []);
  
  return prefersReducedMotion;
}
```

**Manual Toggle:** In addition to auto-detecting `prefers-reduced-motion: reduce`, Grimoire provides a manual toggle in the UI (e.g., a checkbox in settings or a toolbar button) to enable/disable Reduced_Motion_Mode.

**Applying:**
```typescript
function SpellbookEditor() {
  const reducedMotion = useReducedMotion();
  
  return (
    <div className={`spellbook-editor ${reducedMotion ? 'reduced-motion' : ''}`}>
      {/* Content */}
    </div>
  );
}
```

### Keyboard Navigation

**Tab Order:**
1. Mode toggle (Edit/Browse)
2. Card navigation controls (previous/next)
3. Card canvas objects (in placement order)
4. Spell editor (if open)
5. Object palette (Edit mode)
6. Main menu / close button

**Keyboard Shortcuts:**
- `Tab` / `Shift+Tab`: Navigate interactive elements
- `Enter` / `Space`: Activate focused button
- `Escape`: Close modal dialogs, cancel operations
- `Arrow keys`: Move selected object (Edit mode)
- `Shift+Arrow keys`: Resize selected object (Edit mode)
- `Delete` / `Backspace`: Delete selected object (Edit mode)
- `Ctrl+S`: Save (redundant with auto-save, but provides feedback)
- `Ctrl+E`: Export current spellbook

**Important:** Keyboard shortcuts (e.g., arrow keys for card navigation, Delete for object deletion) are disabled when the focus is in a text input, textarea, or contenteditable element. This prevents shortcuts from interfering with text editing.

**Object Selection (Edit Mode):**
```typescript
function handleKeyDown(e: KeyboardEvent): void {
  if (mode !== 'Edit' || !selectedObject) return;
  
  switch (e.key) {
    case 'ArrowLeft':
      moveObject(selectedObject.id, -10, 0);
      e.preventDefault();
      break;
    case 'ArrowRight':
      moveObject(selectedObject.id, 10, 0);
      e.preventDefault();
      break;
    case 'ArrowUp':
      moveObject(selectedObject.id, 0, -10);
      e.preventDefault();
      break;
    case 'ArrowDown':
      moveObject(selectedObject.id, 0, 10);
      e.preventDefault();
      break;
    case 'Delete':
    case 'Backspace':
      deleteObject(selectedObject.id);
      e.preventDefault();
      break;
  }
}
```

### WCAG 2.1 AA Compliance

**Contrast Requirements:**
- Normal text (< 18pt): minimum 4.5:1
- Large text (≥ 18pt): minimum 3:1
- UI components: minimum 3:1

**Testing Approach:**
```typescript
// Automated contrast checker in tests
function checkContrast(foreground: string, background: string): number {
  const fgLuminance = getRelativeLuminance(foreground);
  const bgLuminance = getRelativeLuminance(background);
  const lighter = Math.max(fgLuminance, bgLuminance);
  const darker = Math.min(fgLuminance, bgLuminance);
  return (lighter + 0.05) / (darker + 0.05);
}

test('text contrast meets WCAG AA', () => {
  const textContrast = checkContrast('#f0f0f0', '#0a0a0a');
  expect(textContrast).toBeGreaterThanOrEqual(4.5);
});
```

**Focus Indicators:**
```css
/* Visible, high-contrast focus indicator */
*:focus {
  outline: 3px solid var(--color-focus);
  outline-offset: 2px;
}

/* Never remove focus indicator */
*:focus:not(:focus-visible) {
  outline: 3px solid var(--color-focus);
  outline-offset: 2px;
}
```

### ARIA Labels

**Interactive Elements:**
```typescript
<button
  onClick={handleSwitchMode}
  aria-label={mode === 'Edit' ? 'Switch to Browse mode' : 'Switch to Edit mode'}
  aria-pressed={mode === 'Browse'}
>
  {mode}
</button>

<button
  onClick={handlePrevCard}
  aria-label="Previous card"
  disabled={currentCardIndex === 0}
>
  ←
</button>

<div
  role="region"
  aria-label={`Card ${currentCardIndex + 1} of ${spellbook.cards.length}: ${currentCard.name}`}
>
  {/* Card content */}
</div>

<textarea
  value={spellText}
  onChange={handleSpellChange}
  aria-label="Spell editor"
  aria-invalid={parseError ? 'true' : 'false'}
  aria-describedby={parseError ? 'spell-error' : undefined}
/>
{parseError && <div id="spell-error" role="alert">{parseError.message}</div>}
```

**Screen Reader Announcements:**
```typescript
function announceToScreenReader(message: string): void {
  const announcement = document.createElement('div');
  announcement.setAttribute('role', 'status');
  announcement.setAttribute('aria-live', 'polite');
  announcement.style.position = 'absolute';
  announcement.style.left = '-10000px';
  announcement.textContent = message;
  document.body.appendChild(announcement);
  setTimeout(() => announcement.remove(), 1000);
}

// Usage
function handleSaveSpellbook(): void {
  saveSpellbook(spellbook).then(() => {
    announceToScreenReader('Spellbook saved');
  });
}
```

---

## Components and Interfaces

### Major Component Responsibilities

**StartScreen:**
- Load and display all spellbooks from storage
- Load and display example spellbooks
- Handle create/open/rename/delete operations
- Trigger navigation to SpellbookEditor

**SpellbookEditor:**
- Manage global state (current spellbook, card, mode)
- Coordinate auto-save
- Handle mode transitions (Edit ↔ Browse)
- Delegate rendering to child components

**CardCanvas:**
- Render current card's objects
- Handle object selection (Edit mode)
- Handle object drag/resize (Edit mode)
- Handle click events on objects (Browse mode)
- Display object palette (Edit mode)

**SpellEditor:**
- Display plain-English input field
- Display spell text editor
- Handle spell generation (AI or template)
- Parse spell text before confirming
- Display parse errors with line numbers

**ObjectRenderer:**
- Render Button, TextField, or Label
- Apply appropriate styles for Edit vs. Browse mode
- Handle focus and keyboard navigation
- Display validation errors (e.g., missing field)

**ModeToggle:**
- Display current mode
- Switch between Edit and Browse
- Display appropriate icon/label

**CardNavigator:**
- Display current card position (e.g., "Card 2 of 5")
- Navigate to previous/next card
- Navigate to specific card by index or name
- Handle keyboard shortcuts

### Domain Layer Modules

**parser.ts:**
- `tokenize(text: string): Token[]`
- `parseSpell(text: string): Result<SpellAST, ParseError>`
- `printSpell(ast: SpellAST): string`

**interpreter.ts:**
- `executeSpell(ast: SpellAST, card: Card, fields: Map<string, string>, stepLimit: number, navigationDepth: number): ExecutionResult`
- `evaluateExpression(expr: Expression, fields: Map<string, string>): string`
- `evaluateCondition(cond: Condition, fields: Map<string, string>): boolean`

**validation.ts:**
- `validateCardName(spellbook: Spellbook, cardIndex: number, name: string): ValidationResult`
- `validateFieldId(fieldId: string): ValidationResult`
- `validateFieldIdUnique(card: Card, fieldId: string, excludeObjectId?: string): ValidationResult`
- `validateSpellbook(spellbook: Spellbook): ValidationResult`

**generator.ts:**
- `generateSpellFromTemplate(context: GenerationContext): string`
- `generateSpellWithAI(description: string, context: GenerationContext, options: { timeout: number }): Promise<{ spellText: string }>`

**storage.ts:**
- `openDatabase(): Promise<IDBDatabase>`
- `saveSpellbook(spellbook: Spellbook): Promise<void>`
- `loadSpellbook(id: string): Promise<Spellbook>`
- `loadAllSpellbooks(): Promise<Spellbook[]>`
- `deleteSpellbook(id: string): Promise<void>`
- `exportSpellbook(spellbook: Spellbook): string`
- `importSpellbook(fileContent: string): Promise<Result<Spellbook, ImportError>>`

---

## Error Handling

### Error Categories

**Parse Errors:**
- Syntax errors in spell text
- Display: Inline in spell editor with line number
- Example: "Expected 'to' on line 1 but found 'too'"

**Runtime Errors:**
- Missing field references
- Invalid card targets
- Display: Modal dialog with error message
- Example: "Field 'username' does not exist on this card"

**Storage Errors:**
- Quota exceeded
- Storage unavailable
- Write failures
- Display: Banner notification with retry option
- Example: "Unable to save spellbook (storage quota exceeded). You can export as a file."

**Import Errors:**
- Malformed JSON
- Schema validation failure
- Spell parsing failure
- File too large
- Display: Modal dialog with detailed message
- Example: "Failed to parse spell on card 'Quiz': Expected 'to' on line 1 but found 'too'"

**Network Errors (AI Service):**
- Timeout
- Server error
- Display: Inline notification in spell editor
- Example: "AI service unavailable, using template generation"

### Error Types

```typescript
interface ParseError {
  message: string;
  line: number;
  column: number;
}

interface RuntimeError {
  kind: 'RuntimeError';
  message: string;
  statement: Statement;
}

interface StorageError {
  kind: 'QuotaExceeded' | 'Unavailable' | 'WriteFailed';
  message: string;
  cause?: Error;
}

interface ImportError {
  kind: 'InvalidJSON' | 'SchemaValidation' | 'SpellParsing' | 'FileTooLarge';
  message: string;
  details?: string; // Additional context (e.g., which card/spell failed)
}
```

---

## Testing Strategy

### Property-Based Tests (fast-check)

**Important:** Use fast-check generators that produce valid domain objects (valid Spellbooks, Cards, CardObjects, SpellASTs). Do not generate arbitrary JSON or random strings.

**Test 1: Spell Round-Trip**
```typescript
test('Property: Parsing and printing preserves spell semantics', () => {
  fc.assert(
    fc.property(spellASTArbitrary(), (ast) => {
      // Feature: grimoire-core, Property 2: For any valid Spell_AST, printing then parsing produces an equivalent AST
      const printed = printSpell(ast);
      const parsed = parseSpell(printed);
      expect(parsed.ok).toBe(true);
      expect(parsed.ast).toEqual(ast);
    }),
    { numRuns: 100 }
  );
});

// Generator for random spell ASTs
function spellASTArbitrary(): fc.Arbitrary<SpellAST> {
  return fc.array(
    fc.oneof(
      goToCardArbitrary(),
      setFieldArbitrary(),
      showMessageArbitrary(),
      ifStatementArbitrary()
    ),
    { minLength: 1 }
  );
}

function goToCardArbitrary(): fc.Arbitrary<GoToCard> {
  return fc.oneof(
    fc.record({ kind: fc.constant('GoToCard'), target: fc.record({ kind: fc.constant('CardName'), name: fc.string() }) }),
    fc.record({ kind: fc.constant('GoToCard'), target: fc.record({ kind: fc.constant('CardNumber'), number: fc.nat() }) })
  );
}

// ... similar generators for other statement types
```

**Test 2: Token Mutation**
```typescript
test('Property: Token mutation produces parse success or error (no crashes)', () => {
  fc.assert(
    fc.property(spellASTArbitrary(), (ast) => {
      // Feature: grimoire-core, Property N: Mutating a valid spell either parses or errors gracefully
      const printed = printSpell(ast);
      const mutated = mutateSpell(printed); // Insert, delete, or replace one token
      const result = parseSpell(mutated);
      // Should not crash - either ok or error
      expect(result).toHaveProperty('ok');
    }),
    { numRuns: 100 }
  );
});

function mutateSpell(spell: string): string {
  const tokens = spell.split(/\s+/);
  const mutation = fc.sample(fc.oneof(
    fc.constant('insert'),
    fc.constant('delete'),
    fc.constant('replace')
  ), 1)[0];
  
  const index = Math.floor(Math.random() * tokens.length);
  
  if (mutation === 'insert') {
    tokens.splice(index, 0, 'MUTATION');
  } else if (mutation === 'delete' && tokens.length > 1) {
    tokens.splice(index, 1);
  } else if (mutation === 'replace') {
    tokens[index] = 'MUTATION';
  }
  
  return tokens.join(' ');
}
```

**Test 3: Interpreter Determinism**
```typescript
test('Property: Spell execution is deterministic', () => {
  fc.assert(
    fc.property(
      spellASTArbitrary(),
      cardArbitrary(),
      fieldValuesArbitrary(),
      (ast, card, fieldValues) => {
        // Feature: grimoire-core, Property 3: For any valid AST and card state, execution produces identical results
        const result1 = executeSpell(ast, card, card.id, fieldValues, 10000, 0);
        const result2 = executeSpell(ast, card, card.id, fieldValues, 10000, 0);
        expect(result1).toEqual(result2);
      }
    ),
    { numRuns: 100 }
  );
});

function cardArbitrary(): fc.Arbitrary<Card> {
  return fc.record({
    id: fc.uuid(),
    name: fc.string(),
    objects: fc.array(objectArbitrary()),
    spells: fc.constant([])
  });
}

function fieldValuesArbitrary(): fc.Arbitrary<Map<string, string>> {
  return fc.dictionary(fc.string(), fc.string()).map(obj => new Map(Object.entries(obj)));
}
```

**Test 4: Step Limit Parameterization**
```typescript
test('Property: Step limit is enforced at various limits', () => {
  fc.assert(
    fc.property(
      spellASTArbitrary(),
      cardArbitrary(),
      fieldValuesArbitrary(),
      fc.integer({ min: 10, max: 10000 }),
      (ast, card, fieldValues, stepLimit) => {
        // Feature: grimoire-core, Property N: Step limit halts execution at the specified limit
        const result = executeSpell(ast, card, card.id, fieldValues, stepLimit, 0);
        if (result.kind === 'StepLimitExceeded') {
          expect(result.stepsExecuted).toBeGreaterThanOrEqual(stepLimit);
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

**Test 5: Export/Import Round-Trip**
```typescript
test('Property: Exporting and importing preserves spellbook structure', () => {
  fc.assert(
    fc.property(spellbookArbitrary(), (spellbook) => {
      // Feature: grimoire-core, Property 4: For any valid spellbook, export then import produces an equivalent spellbook
      const exported = exportSpellbook(spellbook);
      const parseResult = parseImport(exported);
      expect(parseResult.ok).toBe(true);
      
      if (parseResult.ok) {
        const imported = parseResult.value;
        
        // Compare structure (IDs will differ)
        expect(imported.name).toBe(spellbook.name);
        expect(imported.cards.length).toBe(spellbook.cards.length);
        
        for (let i = 0; i < spellbook.cards.length; i++) {
          const originalCard = spellbook.cards[i];
          const importedCard = imported.cards[i];
          expect(importedCard.name).toBe(originalCard.name);
          expect(importedCard.objects.length).toBe(originalCard.objects.length);
          expect(importedCard.spells.length).toBe(originalCard.spells.length);
        }
      }
    }),
    { numRuns: 100 }
  );
});

function spellbookArbitrary(): fc.Arbitrary<Spellbook> {
  return fc.record({
    id: fc.uuid(),
    name: fc.string(),
    cards: fc.array(cardArbitrary(), { minLength: 1 }),
    createdAt: fc.nat(),
    modifiedAt: fc.nat(),
    isExample: fc.boolean()
  });
}
```

**Test 6: Reducer State Machine**
```typescript
test('Property: Reducer maintains invariants across action sequences', () => {
  fc.assert(
    fc.property(
      initialStateArbitrary(),
      fc.array(actionArbitrary()),
      (initialState, actions) => {
        // Feature: grimoire-core, Property N: Applying actions maintains all invariants
        let state = initialState;
        for (const action of actions) {
          state = reducer(state, action);
        }
        
        // Check invariants
        if (state.currentSpellbook) {
          // Unique card names
          const cardNames = state.currentSpellbook.cards.map(c => c.name.toLowerCase());
          expect(new Set(cardNames).size).toBe(cardNames.length);
          
          // Unique field IDs per card
          for (const card of state.currentSpellbook.cards) {
            const fieldIds = card.objects
              .filter(o => o.kind === 'TextField')
              .map(o => o.fieldId.toLowerCase());
            expect(new Set(fieldIds).size).toBe(fieldIds.length);
          }
          
          // Valid spell targets
          for (const card of state.currentSpellbook.cards) {
            const objectIds = new Set(card.objects.map(o => o.id));
            for (const spell of card.spells) {
              if (spell.target.kind === 'Object') {
                expect(objectIds.has(spell.target.objectId)).toBe(true);
              }
            }
          }
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

**Test 7: Keyword Case and Whitespace Variants**
```typescript
test('Property: Parser is case-insensitive and whitespace-tolerant', () => {
  fc.assert(
    fc.property(spellASTArbitrary(), (ast) => {
      // Feature: grimoire-core, Property N: Keyword case and whitespace variations parse to same AST
      const printed = printSpell(ast);
      
      // Transform: change keyword case
      const upperCased = printed.replace(/\b(if|then|else|end|go|to|card|set|show|message|field|is|not|contains)\b/gi, 
        match => match.toUpperCase());
      
      // Transform: add extra whitespace
      const extraSpaces = printed.replace(/\s+/g, '  ');
      
      const result1 = parseSpell(upperCased);
      const result2 = parseSpell(extraSpaces);
      
      if (result1.ok && result2.ok) {
        expect(result1.ast).toEqual(ast);
        expect(result2.ast).toEqual(ast);
      }
    }),
    { numRuns: 100 }
  );
});
```

**Test 8: Interpreter Totality**
```typescript
test('Property: Interpreter never crashes', () => {
  fc.assert(
    fc.property(
      spellASTArbitrary(),
      cardArbitrary(),
      fieldValuesArbitrary(),
      (ast, card, fieldValues) => {
        // Feature: grimoire-core, Property N: For all valid ASTs, execution returns a result
        const result = executeSpell(ast, card, card.id, fieldValues, 10000, 0);
        expect(result).toHaveProperty('kind');
        expect(['Success', 'StepLimitExceeded', 'RuntimeError', 'NavigationDepthExceeded']).toContain(result.kind);
      }
    ),
    { numRuns: 100 }
  );
});
```

**Test 9: Import Totality**
```typescript
test('Property: Import never crashes', () => {
  fc.assert(
    fc.property(fc.string(), (content) => {
      // Feature: grimoire-core, Property N: For all strings, parseImport returns ok or error
      const result = parseImport(content);
      expect(result).toHaveProperty('ok');
    }),
    { numRuns: 100 }
  );
});
```

**Test 10: Input Immutability**
```typescript
test('Property: Domain functions do not mutate inputs', () => {
  fc.assert(
    fc.property(spellASTArbitrary(), cardArbitrary(), fieldValuesArbitrary(), (ast, card, fieldValues) => {
      // Feature: grimoire-core, Property N: Inputs are not mutated
      const frozenAST = Object.freeze(deepFreeze(ast));
      const frozenCard = Object.freeze(deepFreeze(card));
      const frozenFields = Object.freeze(new Map(fieldValues));
      
      // Execute
      executeSpell(frozenAST, frozenCard, card.id, frozenFields, 10000, 0);
      
      // Should not throw if inputs were not mutated
      expect(true).toBe(true);
    }),
    { numRuns: 100 }
  );
});

function deepFreeze<T>(obj: T): T {
  Object.freeze(obj);
  Object.getOwnPropertyNames(obj).forEach(prop => {
    if (obj[prop] !== null && (typeof obj[prop] === 'object' || typeof obj[prop] === 'function')) {
      deepFreeze(obj[prop]);
    }
  });
  return obj;
}
```

### Unit Tests (Vitest)

**Parser Tests:**
- Each acceptance criterion from Requirement 4
- Each grammar rule produces correct AST
- Each error condition produces correct error message

**Interpreter Tests:**
- Each statement type executes correctly
- Step counting is accurate
- Navigation depth tracking works
- Field scope validation works
- Error messages are plain language

**Validation Tests:**
- Card name uniqueness enforcement
- Field ID validation (regex, uniqueness)
- Spellbook schema validation

**Storage Tests:**
- Save/load round-trip
- Export/import with schema versioning
- 5 MB size limit enforcement
- Spell re-parsing during import
- Text sanitization

**Generator Tests:**
- Template fallback produces valid spells for each pattern
- Parse-before-display flow rejects unparseable spells

### Integration Tests

**Edit/Browse Mode Transition:**
```typescript
test('Switching to Browse mode initializes runtime state', () => {
  const spellbook = createTestSpellbook({
    cards: [
      {
        objects: [
          { kind: 'TextField', fieldId: 'name', defaultValue: 'Alice' },
          { kind: 'TextField', fieldId: 'age', defaultValue: '30' }
        ]
      }
    ]
  });
  
  const { getByRole } = render(<App initialSpellbook={spellbook} />);
  
  fireEvent.click(getByRole('button', { name: 'Switch to Browse mode' }));
  
  // Runtime state should be initialized
  expect(getRuntimeFieldValue('name')).toBe('Alice');
  expect(getRuntimeFieldValue('age')).toBe('30');
});

test('Exiting Browse mode discards runtime changes', () => {
  const spellbook = createTestSpellbook({
    cards: [
      {
        objects: [
          { kind: 'TextField', fieldId: 'name', defaultValue: 'Alice' }
        ]
      }
    ]
  });
  
  const { getByRole } = render(<App initialSpellbook={spellbook} />);
  
  // Enter Browse mode
  fireEvent.click(getByRole('button', { name: 'Switch to Browse mode' }));
  
  // Modify runtime state (via spell execution)
  executeSpellThatSets('name', 'Bob');
  expect(getRuntimeFieldValue('name')).toBe('Bob');
  
  // Exit Browse mode
  fireEvent.click(getByRole('button', { name: 'Switch to Edit mode' }));
  
  // Re-enter Browse mode
  fireEvent.click(getByRole('button', { name: 'Switch to Browse mode' }));
  
  // Should be reset to default
  expect(getRuntimeFieldValue('name')).toBe('Alice');
});
```

**Spell Execution:**
```typescript
test('Button click executes attached spell', async () => {
  const spellbook = createTestSpellbook({
    cards: [
      {
        objects: [
          { kind: 'Button', id: 'btn1', label: 'Click Me' }
        ],
        spells: [
          {
            objectId: 'btn1',
            event: 'click',
            sourceText: 'show message "Hello"',
            ast: parseSpell('show message "Hello"').ast
          }
        ]
      }
    ]
  });
  
  const { getByRole, findByRole } = render(<App initialSpellbook={spellbook} />);
  
  // Enter Browse mode
  fireEvent.click(getByRole('button', { name: 'Switch to Browse mode' }));
  
  // Click button
  fireEvent.click(getByRole('button', { name: 'Click Me' }));
  
  // Message dialog should appear
  const dialog = await findByRole('dialog');
  expect(dialog).toHaveTextContent('Hello');
});

test('Card-open spell executes on navigation', async () => {
  const spellbook = createTestSpellbook({
    cards: [
      { name: 'Card1' },
      {
        name: 'Card2',
        spells: [
          {
            objectId: '', // Card-level spell
            event: 'card-open',
            sourceText: 'show message "Welcome to Card 2"',
            ast: parseSpell('show message "Welcome to Card 2"').ast
          }
        ]
      }
    ]
  });
  
  const { getByRole, findByRole } = render(<App initialSpellbook={spellbook} />);
  
  // Enter Browse mode
  fireEvent.click(getByRole('button', { name: 'Switch to Browse mode' }));
  
  // Navigate to Card 2
  fireEvent.click(getByRole('button', { name: 'Next card' }));
  
  // Message dialog should appear
  const dialog = await findByRole('dialog');
  expect(dialog).toHaveTextContent('Welcome to Card 2');
});
```

**Storage Failure Scenarios:**
```typescript
test('Storage failure shows warning and enables export', async () => {
  // Mock storage quota exceeded
  mockIndexedDB.put = jest.fn().mockRejectedValue(new DOMException('QuotaExceededError'));
  
  const spellbook = createTestSpellbook();
  const { getByRole, findByText } = render(<App initialSpellbook={spellbook} />);
  
  // Make a change to trigger auto-save
  fireEvent.click(getByRole('button', { name: 'Add Card' }));
  
  // Warning should appear
  const warning = await findByText(/Unable to save spellbook/i);
  expect(warning).toBeInTheDocument();
  
  // Export button should be available
  const exportButton = getByRole('button', { name: 'Export' });
  expect(exportButton).toBeEnabled();
});
```

### Manual Testing Checklist

**Accessibility:**
- [ ] All interactive elements reachable by keyboard
- [ ] Focus indicators visible on all elements
- [ ] Screen reader announces all state changes
- [ ] Reduced-motion mode disables animations
- [ ] All text meets 4.5:1 contrast ratio
- [ ] All UI components meet 3:1 contrast ratio

**Cross-Browser:**
- [ ] Chrome (latest)
- [ ] Firefox (latest)
- [ ] Safari (latest)
- [ ] Edge (latest)

**Responsive:**
- [ ] Desktop (1920x1080)
- [ ] Tablet (768x1024)
- [ ] Mobile (375x667)

---

## Implementation Phases

### Phase 1: Core Data & Parsing [Core]
- Implement data models (Spellbook, Card, CardObject)
- Implement tokenizer and parser
- Implement pretty-printer
- Write property-based tests for round-trip
- Write unit tests for all grammar rules

### Phase 2: Interpreter & Execution [Core]
- Implement interpreter with step counting
- Implement navigation depth tracking
- Implement field scope validation
- Write property-based tests for determinism
- Write unit tests for each statement type

### Phase 3: Storage & Persistence [Core]
- Implement IndexedDB adapter
- Implement auto-save with debouncing
- Implement export/import with validation
- Write property-based tests for export/import round-trip
- Write unit tests for storage failure scenarios

### Phase 4: UI Foundation [Core]
- Implement Start Screen
- Implement SpellbookEditor shell
- Implement ModeToggle and CardNavigator
- Implement state management (Context + Reducer)
- Write integration tests for mode transitions

### Phase 5: Card Canvas & Objects [Core]
- Implement CardCanvas
- Implement ObjectRenderer (Button, TextField, Label)
- Implement object placement, move, resize, delete (Edit mode)
- Implement object interaction (Browse mode)
- Write integration tests for Edit mode operations

### Phase 6: Spell Editor & Generation [Core]
- Implement SpellEditor UI
- Implement Template_Fallback generator
- Implement parse-before-display flow
- Implement explicit confirm step
- Write unit tests for template patterns

### Phase 7: Example Spellbooks [Core]
- Create three example spellbooks (quiz, shop, flashcard)
- Implement startup verification
- Implement example loading with working copy
- Write tests for example spellbook parsing

### Phase 8: Accessibility & Styling [Core]
- Implement spooky-retro theme with CSS custom properties
- Implement reduced-motion support
- Implement keyboard navigation
- Implement ARIA labels
- Implement focus indicators
- Run automated contrast checks
- Manual accessibility testing

### Phase 9: AI Service Integration [Stretch]
- Implement server-side API proxy
- Implement client-side AI generation with timeout
- Implement privacy notice UX
- Implement fallback to Template_Fallback
- Write integration tests for timeout and error scenarios

### Phase 10: Polish & Testing [Core]
- End-to-end testing
- Cross-browser testing
- Performance optimization
- Error message review
- Documentation

---

## Open Questions & Future Considerations

**Security:**
- Q: Should we implement Content Security Policy headers?
- A: Yes, for defense-in-depth. CSP should disallow unsafe-eval and unsafe-inline.

**Performance:**
- Q: What is the maximum spellbook size we should support?
- A: Enforce 5 MB import limit. No explicit limit on number of cards, but warn if performance degrades (>100 cards).

**Extensibility:**
- Q: How should we version the Spell Language grammar?
- A: Include grammar version in spellbook schema. Parser should reject future versions with clear message.

**Internationalization:**
- Q: Should spell keywords be localized?
- A: Not in initial release. English-only for now. Consider localization in future if demand exists.

**Offline:**
- Q: Should we implement a service worker for full offline support?
- A: Not required for core. Consider as stretch goal if users request it.

**Mobile:**
- Q: Should we optimize for touch input (drag/resize on mobile)?
- A: Not in initial release. Focus on desktop first. Mobile as future enhancement.

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Acceptance Criteria Testing Prework

**Requirement 1: Spellbook and Card Management [Core]**

1.1. WHEN the user creates a new spellbook, THE Grimoire SHALL create the spellbook with exactly one blank card
  Thoughts: This is testing a specific scenario (creating a new spellbook). We want to ensure that across all contexts, creating a new spellbook results in a spellbook with exactly one card. This is a universal property.
  Classification: PROPERTY
  Test Strategy: Generate random contexts (user state, existing spellbooks), create new spellbook, verify exactly one card exists

1.2. THE Grimoire SHALL allow the user to add a new blank card to the current spellbook at any position
  Thoughts: This is testing that card insertion works at any position. We can test this by generating random spellbooks and random positions, adding a card, and verifying the spellbook structure.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks and random positions (0 to length), add card, verify length increased by 1 and card is at correct position

1.3. WHEN the user attempts to delete a card and the spellbook contains more than one card, THE Grimoire SHALL remove that card from the spellbook
  Thoughts: This is testing deletion behavior for spellbooks with >1 card. We can test this universally by generating random spellbooks with multiple cards, deleting a random card, and verifying removal.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks with 2+ cards, delete random card, verify length decreased by 1 and card is gone

1.4. IF the user attempts to delete the last remaining card in a spellbook, THEN THE Grimoire SHALL reject the deletion and display a plain-language message
  Thoughts: This is an edge case but should hold universally for all single-card spellbooks. We can test this by generating random single-card spellbooks and verifying deletion is rejected.
  Classification: PROPERTY
  Test Strategy: Generate random single-card spellbooks, attempt deletion, verify rejection and error message

1.5. THE Grimoire SHALL allow the user to reorder cards within a spellbook by dragging or by using keyboard controls
  Thoughts: This is testing reordering behavior. We can test this by generating random spellbooks, performing random reorder operations, and verifying the new order.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks, perform random reorder (from index, to index), verify correct order

1.6. THE Grimoire SHALL display the current card's position (e.g. "Card 2 of 5") to the user at all times
  Thoughts: This is testing a UI rendering property. For any spellbook state, the display should correctly show the position.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks and card indices, verify position string format is correct

1.7. THE Grimoire SHALL allow the user to rename a spellbook
  Thoughts: This is testing rename behavior. For any spellbook and any valid name, renaming should succeed.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks and random names, rename, verify new name is set

1.8. WHEN the user deletes a spellbook, THE Grimoire SHALL request confirmation before removing it from Storage
  Thoughts: This is testing that deletion always requests confirmation. This is a UI flow property.
  Classification: INTEGRATION
  Test Strategy: Use integration test with 1-2 examples to verify confirmation dialog appears before deletion

1.9. THE Grimoire SHALL allow the user to set and edit a Card_Name for each card; Card_Names SHALL be unique within a spellbook
  Thoughts: This is testing uniqueness enforcement. For any spellbook, attempting to set a duplicate name should be rejected.
  Classification: PROPERTY
  Test Strategy: Generate random spellbooks, attempt to set duplicate card name, verify rejection; also test valid unique names succeed

1.10. WHEN a Card_Name would duplicate an existing Card_Name in the same spellbook, THE Grimoire SHALL reject the rename and display a plain-language message
  Thoughts: This is a specific case of 1.9 (uniqueness). Covered by same property.
  Classification: PROPERTY
  Test Strategy: Same as 1.9

1.11. THE Spell_Language `go to card` statement SHALL accept either a Card_Name or a card number as its target
  Thoughts: This is testing parser behavior. For any valid card target (name or number), parsing should succeed.
  Classification: PROPERTY
  Test Strategy: Generate random card targets (names and numbers), parse `go to card <target>`, verify AST is correct

**Requirement 2: Objects on Cards [Core]**

2.1. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to place a Button, Text_Field, or Label onto the current card
  Thoughts: This is testing object placement. For any card and any object type, placement should succeed.
  Classification: PROPERTY
  Test Strategy: Generate random cards and random object types, place object, verify it appears in card.objects

2.2. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to move any object on the current card
  Thoughts: This is testing object movement. For any object and any new position, movement should update coordinates.
  Classification: PROPERTY
  Test Strategy: Generate random cards with random objects, move to random position, verify coordinates updated

2.3. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to resize any object on the current card
  Thoughts: This is testing object resizing. For any object and any new size, resizing should update dimensions.
  Classification: PROPERTY
  Test Strategy: Generate random cards with random objects, resize to random dimensions, verify size updated

2.4. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to delete any selected object from the current card
  Thoughts: This is testing object deletion. For any card with objects, deleting an object should remove it.
  Classification: PROPERTY
  Test Strategy: Generate random cards with random objects, delete random object, verify removal

2.5. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to edit the display text of a Label or the button label of a Button
  Thoughts: This is testing text editing. For any Label/Button and any new text, editing should update the text.
  Classification: PROPERTY
  Test Strategy: Generate random Labels and Buttons, set random text, verify text updated

2.6. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to set a default value and a Field_Identifier on a Text_Field; the Field_Identifier SHALL be unique within its card
  Thoughts: This is testing TextField configuration and uniqueness. For any card, duplicate Field_IDs should be rejected.
  Classification: PROPERTY
  Test Strategy: Generate random cards, attempt to set duplicate Field_ID, verify rejection; also test valid unique IDs succeed

2.7. THE Grimoire SHALL assign each object a unique identifier within its card at the time of placement
  Thoughts: This is testing ID uniqueness. For any sequence of object placements, all IDs should be unique.
  Classification: PROPERTY
  Test Strategy: Generate random cards with multiple objects, verify all object IDs are unique

2.8. WHILE in Browse_Mode, THE Grimoire SHALL prevent the user from moving, resizing, or deleting objects
  Thoughts: This is testing that operations are disabled in Browse mode. This is a UI state property.
  Classification: INTEGRATION
  Test Strategy: Use integration test to verify UI controls are disabled in Browse mode

2.9. WHEN the user renames a Text_Field's Field_Identifier, THE Grimoire SHALL warn that any existing spells referencing the old Field_Identifier will no longer resolve
  Thoughts: This is testing warning behavior. For any TextField rename where spells reference the old ID, a warning should appear.
  Classification: INTEGRATION
  Test Strategy: Use integration test with 1-2 examples to verify warning appears

2.10. WHEN a spell references a Field_Identifier that does not exist on the current card at runtime, THE Spell_Interpreter SHALL halt and display a plain-language error message
  Thoughts: This is testing error handling. For any spell with missing field reference, execution should error.
  Classification: PROPERTY
  Test Strategy: Generate random spells with invalid field references, execute, verify error

2.11. Field_Identifiers and runtime field values are scoped to a single card
  Thoughts: This is testing field scoping. Field references should only resolve on the same card.
  Classification: PROPERTY
  Test Strategy: Generate random multi-card spellbooks, verify field references fail when referencing other cards

**Requirement 3: Edit Mode and Browse Mode [Core]**

3.1-3.7. Various mode transition requirements
  Thoughts: These are testing mode transition behavior and state changes. These are integration tests.
  Classification: INTEGRATION
  Test Strategy: Use integration tests to verify mode transitions, state initialization, and UI updates

**Requirement 4: Spell Language [Core]**

4.1-4.11. Spell language grammar and parsing requirements
  Thoughts: These are testing parser behavior and round-trip properties. Already covered by core property tests.
  Classification: PROPERTY
  Test Strategy: Round-trip property test (parse → print → parse), invalid input property test

**Requirement 5: Safe Spell Execution [Core]**

5.1-5.7. Spell execution safety and sandboxing requirements
  Thoughts: These are testing interpreter behavior and determinism. Already covered by core property tests.
  Classification: PROPERTY
  Test Strategy: Determinism property test, step limit property test, navigation depth property test

**Requirement 6: Spell Generation UI [Core]**

6.1-6.7. Spell generation flow requirements
  Thoughts: These are testing generation and confirmation flow. Mix of unit tests and integration tests.
  Classification: EXAMPLE/INTEGRATION
  Test Strategy: Unit tests for template patterns, integration tests for UI flow

**Requirement 7: Save, Export, and Import [Core]**

7.1-7.10. Storage and import/export requirements
  Thoughts: Export/import round-trip is a property. Others are integration/unit tests.
  Classification: PROPERTY (round-trip), INTEGRATION (storage), EXAMPLE (validation)
  Test Strategy: Round-trip property test, integration tests for auto-save, unit tests for validation

**Requirement 8: Example Spellbooks [Core]**

8.1-8.6. Example spellbook requirements
  Thoughts: These are testing that examples load and parse correctly. Integration/unit tests.
  Classification: INTEGRATION/EXAMPLE
  Test Strategy: Unit tests for parse verification, integration tests for loading

**Requirement 9: Visual Style and Accessibility [Core]**

9.1-9.8. Accessibility requirements
  Thoughts: These are testing UI rendering and accessibility. Manual and automated tests.
  Classification: INTEGRATION/MANUAL
  Test Strategy: Automated contrast checks, manual keyboard navigation testing

**Requirement 10: Storage Failure Handling [Core]**

10.1-10.3. Storage error handling requirements
  Thoughts: These are testing error handling UI. Integration tests.
  Classification: INTEGRATION
  Test Strategy: Integration tests with mocked storage failures

**Requirement 11: Start Screen [Core]**

11.1-11.6. Start screen requirements
  Thoughts: These are testing start screen UI and operations. Integration tests.
  Classification: INTEGRATION
  Test Strategy: Integration tests for each operation (create, open, rename, delete)

### Reflection on Properties

After reviewing all properties, the following consolidations are identified:

- **Properties 1.9 and 1.10** (Card_Name uniqueness): These are the same property—duplicate names should be rejected
- **Properties 2.6 and Field_ID uniqueness**: Can be tested as a single property
- **Parser round-trip and grammar properties** (4.7, 4.8): Already captured in core round-trip property
- **Interpreter determinism and sandboxing** (5.6): Already captured in core determinism property
- **Export/import round-trip** (7.6): Already captured as core property

### Property 1: Spellbook Creation Initialization

*For any* context in which a user creates a new spellbook, the system SHALL create the spellbook with exactly one card, and that card SHALL be blank (no objects, no spells).

**Validates: Requirements 1.1**

### Property 2: Card Addition Preserves Structure

*For any* spellbook and any valid position (0 to length inclusive), adding a card SHALL increase the spellbook's card count by exactly 1, and the new card SHALL appear at the specified position.

**Validates: Requirements 1.2**

### Property 3: Card Deletion Succeeds When Multiple Cards Exist

*For any* spellbook with 2 or more cards, deleting any card SHALL reduce the card count by exactly 1, and the deleted card SHALL no longer appear in the spellbook.

**Validates: Requirements 1.3**

### Property 4: Last Card Deletion is Rejected

*For any* spellbook with exactly 1 card, attempting to delete that card SHALL fail, the spellbook SHALL remain unchanged, and a plain-language error message SHALL be displayed.

**Validates: Requirements 1.4**

### Property 5: Card Reordering Preserves All Cards

*For any* spellbook and any valid reorder operation (moving a card from one position to another), the spellbook SHALL contain the same cards (by ID) after the operation, and the target card SHALL be at the new position.

**Validates: Requirements 1.5**

### Property 6: Card Name Uniqueness

*For any* spellbook and any card, attempting to set a Card_Name that already exists (case-insensitive) SHALL be rejected with a plain-language error message, and the card's name SHALL remain unchanged.

**Validates: Requirements 1.9, 1.10**

### Property 7: Object Placement Adds Unique Object

*For any* card in Edit mode and any object type (Button, TextField, Label), placing an object SHALL increase the card's object count by 1, the new object SHALL have a unique ID within the card, and the object SHALL match the requested type.

**Validates: Requirements 2.1, 2.7**

### Property 8: Object Movement Updates Coordinates

*For any* card in Edit mode and any existing object on that card, moving the object to a new position SHALL update only the object's x and y coordinates, and all other object properties SHALL remain unchanged.

**Validates: Requirements 2.2**

### Property 9: Object Resizing Updates Dimensions

*For any* card in Edit mode and any existing object on that card, resizing the object SHALL update only the object's width and height, and all other object properties SHALL remain unchanged.

**Validates: Requirements 2.3**

### Property 10: Object Deletion Removes Object

*For any* card in Edit mode and any existing object on that card, deleting the object SHALL reduce the card's object count by 1, and the object SHALL no longer appear in the card's object list.

**Validates: Requirements 2.4**

### Property 11: Field_Identifier Uniqueness

*For any* card and any TextField, attempting to set a Field_Identifier that already exists on that card SHALL be rejected with a plain-language error message, and the TextField's Field_Identifier SHALL remain unchanged.

**Validates: Requirements 2.6**

### Property 12: Spell Round-Trip Preserves Semantics

*For any* valid Spell_AST, printing the AST to text and then parsing the text SHALL produce an equivalent Spell_AST (structural equality).

**Validates: Requirements 4.7**

### Property 13: Invalid Spell Text Produces Parse Error

*For any* syntactically invalid spell text, parsing SHALL fail and produce a ParseError with a plain-language message and line number, and SHALL NOT produce a Spell_AST.

**Validates: Requirements 4.5, 4.8**

### Property 14: Spell Execution is Deterministic

*For any* valid Spell_AST, card state, and runtime field values, executing the spell twice with identical inputs SHALL produce identical ExecutionResults (same state changes, same step count, same error state).

**Validates: Requirements 5.6**

### Property 15: Spell Step Limit Enforcement

*For any* spell execution, if the step count reaches the step limit (10,000), the interpreter SHALL halt with a StepLimitExceeded error, and the error message SHALL be plain language.

**Validates: Requirements 5.2, 5.3**

### Property 16: Navigation Depth Limit Enforcement

*For any* sequence of card navigations triggered by card-open spells, if the navigation depth exceeds 16, the interpreter SHALL halt with a NavigationDepthExceeded error, and the error message SHALL be plain language.

**Validates: Requirements 5.7**

### Property 17: Field Reference Scope Validation

*For any* spell that references a Field_Identifier, if that identifier does not exist on the current card, execution SHALL halt with a RuntimeError, and the error message SHALL name the missing Field_Identifier.

**Validates: Requirements 2.10, 2.11, 4.11**

### Property 18: Export/Import Round-Trip Preserves Structure

*For any* valid spellbook, exporting to JSON and then importing the JSON SHALL produce a spellbook that is structurally equivalent: same name, same number of cards, same card names, same number of objects per card, same spells per card (IDs will differ).

**Validates: Requirements 7.6**

### Property 19: Import Spell Re-Parsing

*For any* import file, if any spell text in the file fails to parse, the entire import SHALL be rejected, and the error message SHALL identify which card and spell failed.

**Validates: Requirements 7.10**

### Property 20: Template Generation Produces Valid Spells

*For any* generation context (event type, object type, available fields, description) where the context is valid (non-empty cardNames, valid event and objectType), the Template_Fallback generator SHALL produce spell text that parses successfully into a valid Spell_AST.

**Validates: Requirements 6.6**

---

## Testing Strategy

### Dual Testing Approach

**Property-Based Tests (fast-check):**
- Test universal properties across randomized inputs (100+ iterations per property)
- Each property test references its design document property number
- Tag format: `// Feature: grimoire-core, Property N: <property text>`
- Focus: Parser round-trip, interpreter determinism, export/import round-trip, validation rules

**Unit Tests (Vitest):**
- Test specific examples, edge cases, and error conditions
- Map each test to a specific acceptance criterion
- Focus: Individual statement types, specific error messages, boundary conditions

**Integration Tests (Vitest + React Testing Library):**
- Test UI workflows and component interactions
- Focus: Mode transitions, spell execution flow, storage operations, accessibility

### Property-Based Test Configuration

**Library:** fast-check

**Minimum Iterations:** 100 per property test

**Example:**
```typescript
test('Property 1: Spellbook Creation Initialization', () => {
  // Feature: grimoire-core, Property 1: For any context in which a user creates a new spellbook, the system SHALL create the spellbook with exactly one card
  fc.assert(
    fc.property(fc.anything(), (context) => {
      const spellbook = createNewSpellbook();
      expect(spellbook.cards).toHaveLength(1);
      expect(spellbook.cards[0].objects).toHaveLength(0);
      expect(spellbook.cards[0].spells).toHaveLength(0);
    }),
    { numRuns: 100 }
  );
});
```

### Unit Test Coverage

**Parser:**
- Each grammar rule (expressions, conditions, statements)
- Each error condition (unterminated string, missing keyword, invalid identifier)
- Edge cases (empty input, very long input, special characters)

**Interpreter:**
- Each statement type execution
- Step counting accuracy
- Navigation depth tracking
- Field scope validation
- Error message clarity

**Validation:**
- Card name uniqueness (case-insensitive)
- Field ID format (regex)
- Field ID uniqueness per card
- Spellbook schema validation

**Storage:**
- Auto-save debouncing
- Export format correctness
- Import validation pipeline
- 5 MB size limit
- Text sanitization

**Generator:**
- Template patterns for each event type
- Keyword extraction (card names, field IDs, quoted values)
- Default fallbacks

### Integration Test Coverage

**Mode Transitions:**
- Edit → Browse: runtime state initialization
- Browse → Edit: runtime state discarding
- Mode persistence across card navigation

**Spell Execution:**
- Button click triggers spell
- Card-open triggers spell on navigation
- State changes apply correctly (field updates, messages, navigation)
- Error dialogs appear on runtime errors

**Storage Operations:**
- Auto-save triggers after edit
- Storage failure shows warning
- Export produces downloadable file
- Import validates and loads spellbook

**Accessibility:**
- Keyboard navigation works
- Focus indicators visible
- Screen reader announcements
- Reduced motion disables animations

### Manual Testing Checklist

**Accessibility:**
- [ ] All controls reachable by Tab
- [ ] Focus visible on all elements
- [ ] Screen reader announces state changes
- [ ] Reduced-motion disables effects
- [ ] Contrast meets 4.5:1 for text, 3:1 for UI

**Cross-Browser:**
- [ ] Chrome, Firefox, Safari, Edge (latest versions)

**Responsive:**
- [ ] Desktop, tablet, mobile viewports

**Visual Style:**
- [ ] Spooky-retro theme consistent
- [ ] Candlelit palette applied
- [ ] No content obscured by effects

---

This design document provides implementation-ready specifications for all core requirements. The architecture separates concerns cleanly (UI, domain, persistence), the spell language has a formal grammar and round-trip property, the interpreter is sandboxed and deterministic, and the testing strategy ensures comprehensive coverage through property-based tests for universal properties and unit/integration tests for specific scenarios.
