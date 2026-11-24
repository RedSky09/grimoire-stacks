# Requirements Document

## Introduction

Grimoire is a browser-based tool — tagline: "build small apps from cards, in plain English" — that lets non-programmers create small interactive applications from stacks of cards called spellbooks. Users place objects (buttons, text fields, labels) on cards, attach short English-like scripts called spells to object events, and switch between edit mode (authoring) and browse mode (running) without leaving the browser. Grimoire ships three built-in example spellbooks (a quiz, a shop price list, and a flashcard deck) and persists user spellbooks in the browser with export/import support.

**Core vs. Stretch:** Requirements marked **[Core]** must be implemented for a usable first release. Requirements marked **[Stretch]** are valuable but may be deferred to a later iteration.

---

## Glossary

- **Grimoire**: The application described in this document.
- **Spellbook**: A named, ordered collection of cards. Equivalent to a "stack" in internal code.
- **Card**: A single page-sized canvas within a spellbook that holds objects.
- **Card_Name**: A user-editable string that identifies a card within its spellbook. Card_Names must be unique within a spellbook.
- **Object**: A UI widget placed on a card — one of Button, Text_Field, or Label.
- **Field_Identifier**: A non-empty alphanumeric string (letters, digits, and underscores; no spaces; not starting with a digit) that uniquely identifies a Text_Field within its card.
- **Spell**: A short script in the Spell Language attached to an object event.
- **Spell_Language**: The small, English-like scripting language defined in Requirement 4.
- **Spell_Expression**: A value-producing term in the Spell_Language — either a quoted string literal, a number literal, or a `field <id>` read.
- **Spell_Condition**: A boolean test in the Spell_Language — `<expression> is <expression>`, `<expression> is not <expression>`, or `<expression> contains <expression>`.
- **Spell_AST**: The abstract syntax tree produced by parsing a Spell.
- **Spell_Interpreter**: The sandboxed runtime that executes a Spell_AST.
- **Edit_Mode**: The application mode in which the user authors spellbooks, cards, and objects.
- **Browse_Mode**: The application mode in which spells run in response to user interaction.
- **Spell_Editor**: The UI component in which users write or view a Spell.
- **Spell_Generator**: The subsystem that takes a plain-English description and proposes a Spell.
- **AI_Service**: The external server-side service that processes plain-English descriptions and returns candidate Spell text.
- **Template_Fallback**: A rule-based Spell_Generator used when the AI_Service is unavailable.
- **Storage**: The browser's localStorage or IndexedDB used to persist spellbooks.
- **Export_File**: A JSON file produced by exporting a spellbook.
- **Import_File**: A JSON file the user supplies to restore a spellbook.
- **Step_Limit**: The maximum number of interpreter steps allowed per Spell execution.
- **Reduced_Motion_Mode**: A display mode that disables non-essential animations.

---

## Requirements

### Requirement 1: Spellbook and Card Management [Core]

**User Story:** As a non-programmer, I want to create and organise spellbooks made of cards, so that I can structure my app across multiple screens.

#### Acceptance Criteria

1. WHEN the user creates a new spellbook, THE Grimoire SHALL create the spellbook with exactly one blank card.
2. THE Grimoire SHALL allow the user to add a new blank card to the current spellbook at any position.
3. WHEN the user attempts to delete a card and the spellbook contains more than one card, THE Grimoire SHALL remove that card from the spellbook.
4. IF the user attempts to delete the last remaining card in a spellbook, THEN THE Grimoire SHALL reject the deletion and display a plain-language message explaining that a spellbook must keep at least one card.
5. THE Grimoire SHALL allow the user to reorder cards within a spellbook by dragging or by using keyboard controls.
6. THE Grimoire SHALL display the current card's position (e.g. "Card 2 of 5") to the user at all times.
7. THE Grimoire SHALL allow the user to rename a spellbook.
8. WHEN the user deletes a spellbook, THE Grimoire SHALL request confirmation before removing it from Storage.
9. THE Grimoire SHALL allow the user to set and edit a Card_Name for each card; Card_Names SHALL be unique within a spellbook.
10. WHEN a Card_Name would duplicate an existing Card_Name in the same spellbook, THE Grimoire SHALL reject the rename and display a plain-language message indicating the conflict.
11. THE Spell_Language `go to card` statement SHALL accept either a Card_Name or a card number as its target.

---

### Requirement 2: Objects on Cards [Core]

**User Story:** As a non-programmer, I want to place, move, resize, and delete buttons, text fields, and labels on cards, so that I can design the layout of each screen.

#### Acceptance Criteria

1. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to place a Button, Text_Field, or Label onto the current card.
2. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to move any object on the current card by dragging or by keyboard arrow keys.
3. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to resize any object on the current card by dragging its resize handles or by keyboard controls.
4. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to delete any selected object from the current card.
5. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to edit the display text of a Label or the button label of a Button.
6. WHILE in Edit_Mode, THE Grimoire SHALL allow the user to set a default value and a Field_Identifier on a Text_Field; the Field_Identifier SHALL be a non-empty alphanumeric string (letters, digits, and underscores; not starting with a digit) and SHALL be unique within its card.
7. THE Grimoire SHALL assign each object a unique identifier within its card at the time of placement.
8. WHILE in Browse_Mode, THE Grimoire SHALL prevent the user from moving, resizing, or deleting objects.
9. WHEN the user renames a Text_Field's Field_Identifier, THE Grimoire SHALL warn that any existing spells referencing the old Field_Identifier will no longer resolve, and SHALL display those spells to the user.
10. WHEN a spell references a Field_Identifier that does not exist on the current card at runtime, THE Spell_Interpreter SHALL halt and display a plain-language error message naming the missing Field_Identifier.
11. Field_Identifiers and runtime field values are scoped to a single card; a spell SHALL only read or set fields on the card it is currently running on, and SHALL NOT reference fields on any other card.

---

### Requirement 3: Edit Mode and Browse Mode [Core]

**User Story:** As a non-programmer, I want to switch between editing my spellbook layout and running it, so that I can test my app without leaving Grimoire.

#### Acceptance Criteria

1. THE Grimoire SHALL provide a clearly labelled control to switch between Edit_Mode and Browse_Mode.
2. WHEN the user switches to Browse_Mode, THE Grimoire SHALL apply all spells attached to objects and render the card in its runtime state.
3. WHEN the user switches to Edit_Mode from Browse_Mode, THE Grimoire SHALL restore the authoring view without altering any spells or object positions.
4. WHILE in Browse_Mode, THE Grimoire SHALL prevent structural modifications to cards and objects (placement, deletion, resize, move).
5. WHILE in Edit_Mode, THE Grimoire SHALL not execute spells in response to user interaction with objects.
6. THE Grimoire SHALL preserve the current card position when the user switches modes.
7. WHEN the user enters Browse_Mode, THE Grimoire SHALL initialise all Text_Field runtime values from the Text_Field's authored default value (an empty string if no default has been set by the author); spells MAY read and set these runtime values during Browse_Mode, but THE Grimoire SHALL never modify the authored content of any object; WHEN the user returns to Edit_Mode, THE Grimoire SHALL discard all runtime field values.
8. WHEN the user enters Browse_Mode, THE Grimoire SHALL execute the `card-open` spell for the current card (if one exists) with a navigation depth of 0.

---

### Requirement 4: Spell Language [Core]

**User Story:** As a non-programmer, I want to write short English-like scripts attached to object events, so that I can give my app interactive behavior.

#### Acceptance Criteria

1. THE Spell_Language SHALL support the following expressions, conditions, and statements:
   - **Expressions (Spell_Expression):** a quoted string literal, a number literal, or `field <id>` (reads the current runtime value of the Text_Field identified by `<id>`).
   - **Conditions (Spell_Condition):** `<expression> is <expression>`, `<expression> is not <expression>`, or `<expression> contains <expression>`.
   - **Statements:**
     - A spell consists of one or more statements, one per line
     - `go to card <name or number>` — navigate to a card by Card_Name (quoted string) or card number (integer), and terminate spell execution; any statements after `go to card` are unreachable
     - `set <field-id> to <expression>` — write the value of a Spell_Expression into the Text_Field identified by `<field-id>`
     - `show message <expression>` — display a modal message whose text is the value of a Spell_Expression
     - `if <condition> then` ... `end if` — conditional branch on a Spell_Condition; the block form is:
       ```
       if <condition> then
         <statement>
         ...
       else
         <statement>
         ...
       end if
       ```
       The `else` block is optional; the `end if` is required
2. THE Spell_Language SHALL reserve the following keywords (case-insensitive): `go`, `to`, `card`, `set`, `show`, `message`, `if`, `then`, `else`, `end`, `field`, `is`, `not`, `contains`. Field_Identifiers SHALL NOT use any reserved keyword.
3. String literals in the Spell_Language SHALL support escape sequences: `\"` (literal quote), `\\` (literal backslash), and `\n` (newline).
4. Card names in `go to card "<name>"` are matched case-insensitively at runtime.
5. THE Grimoire SHALL allow a Spell to be attached to the `click` event of a Button.
6. THE Grimoire SHALL allow a Spell to be attached to the `card-open` event of a Card.
7. WHEN the Spell_Editor is submitted with syntactically valid Spell text, THE Grimoire SHALL parse the text into a Spell_AST and attach it to the target event.
8. WHEN the Spell_Editor is submitted with syntactically invalid Spell text, THE Grimoire SHALL display a plain-language error message that identifies the line number of the first error and does not attach any Spell.
9. THE Grimoire SHALL define the complete grammar of the Spell_Language in its technical documentation, and THE Spell_Language parser SHALL accept all inputs conforming to that grammar.
10. FOR ALL valid Spell texts, parsing the text, printing the resulting Spell_AST back to text, and parsing again SHALL produce an equivalent Spell_AST (round-trip property).
11. FOR ALL syntactically invalid Spell texts, THE Grimoire SHALL produce a parse error rather than a Spell_AST (error-condition property).
12. THE Spell_Language SHALL compare all values as text strings; number literals are stored as their text representation for the purpose of comparisons.
13. The `contains` condition SHALL test whether the left-hand text string contains the right-hand text string as a substring (case-sensitive).
14. A Spell_Expression using `field <id>` SHALL only resolve identifiers that exist on the card to which the spell is attached; referencing a Field_Identifier from another card is a runtime error.

---

### Requirement 5: Safe Spell Execution [Core]

**User Story:** As a non-programmer, I want spells to run safely so that a badly-written spell cannot break my browser or loop forever.

#### Acceptance Criteria

1. THE Spell_Interpreter SHALL execute only a Spell_AST; THE Grimoire SHALL never evaluate raw spell text as code.
2. THE Spell_Interpreter SHALL enforce a Step_Limit of 10,000 interpreter steps per Spell execution.
3. WHEN a Spell execution reaches the Step_Limit, THE Spell_Interpreter SHALL halt execution and display a plain-language message to the user: "This spell was stopped because it performed too many steps." Navigation chains triggered by card-open spells are the primary source of runaway execution and are separately limited by criterion 5.7.
4. THE Spell_Interpreter SHALL have no access to the browser DOM, network APIs, or browser storage during execution.
5. WHEN a runtime error occurs during Spell execution (e.g. referencing a non-existent field), THE Spell_Interpreter SHALL halt execution and display a plain-language error message identifying the statement that caused the error.
6. FOR ALL valid Spell_ASTs, executing the same Spell_AST with the same card state and the same input field values SHALL produce the same set of state changes and messages (determinism property).
7. THE Spell_Interpreter SHALL enforce a maximum card-navigation chain depth of 16; WHEN a card-open spell triggers navigation that would exceed this depth, THE Spell_Interpreter SHALL halt and display a plain-language message to the user.
8. WHEN a spell executes a `go to card` statement and navigates to a target card, THE Grimoire SHALL execute the `card-open` spell for the target card (if one exists), incrementing the navigation depth by 1 before executing the card-open spell.
9. Runtime field values are keyed by card and field identifier; a spell reading or setting a field SHALL only access fields on the card it is currently running on.

---

### Requirement 6: Spell Generation UI [Core]

**User Story:** As a non-programmer, I want to describe what I want a spell to do in plain English and have Grimoire propose it for me, so that I can author spells without learning the syntax.

#### Acceptance Criteria

1. WHILE in Edit_Mode, THE Grimoire SHALL provide a plain-English description input in the Spell_Editor for any object event.
2. WHEN the user submits a plain-English description, THE Spell_Generator SHALL produce a candidate Spell text and display it in the Spell_Editor before attaching it.
3. THE Grimoire SHALL not attach a generated Spell until the user explicitly confirms it.
4. WHEN the Spell_Generator produces a candidate Spell text, THE Grimoire SHALL parse the text into a Spell_AST before displaying it; IF parsing fails, THEN THE Grimoire SHALL show a generation-failure message rather than display unparseable text.
5. IF the AI_Service is unavailable or returns an error, THEN THE Grimoire SHALL use the Template_Fallback to generate a Spell and indicate to the user that the AI_Service was unavailable.
6. THE Template_Fallback SHALL produce syntactically valid Spell text for all supported event types.
7. THE Grimoire SHALL allow the user to edit the candidate Spell in the Spell_Editor before confirming.

---

### Requirement 12: AI Service Integration [Stretch]

**User Story:** As a non-programmer, I want Grimoire to use an AI service to convert my plain-English descriptions into spells, so that the suggestions are more accurate and context-aware.

#### Acceptance Criteria

1. THE Grimoire SHALL never include AI_Service API keys or credentials in browser-side code or expose them to the client.
2. WHEN a plain-English description is submitted, THE Grimoire SHALL display a notice to the user that the description will be sent to a server for processing.
3. THE Grimoire SHALL enforce a request timeout of no more than 10 seconds for AI_Service calls; IF the request times out or returns an error, THEN THE Grimoire SHALL fall back to the Template_Fallback.

---

### Requirement 7: Save, Export, and Import [Core]

**User Story:** As a non-programmer, I want my spellbooks to be saved automatically and to be able to share them as files, so that I do not lose my work and can move spellbooks between devices.

#### Acceptance Criteria

1. THE Grimoire SHALL automatically save all spellbook changes to Storage without requiring a manual save action from the user.
2. WHEN the user exports a spellbook, THE Grimoire SHALL write the spellbook's complete data to an Export_File in JSON format and prompt the browser to download it.
3. WHEN the user imports an Import_File, THE Grimoire SHALL parse and validate the file against the spellbook schema before adding it to Storage.
4. IF an Import_File is malformed or fails schema validation, THEN THE Grimoire SHALL display a plain-language error message describing the problem and leave all existing spellbooks in Storage unchanged.
5. WHEN a valid Import_File is imported, THE Grimoire SHALL add it as a new spellbook without overwriting any existing spellbook.
6. FOR ALL spellbooks S, exporting S and then importing the resulting Export_File SHALL produce a spellbook that is structurally equivalent to S (round-trip property).
7. THE Grimoire SHALL validate every Import_File against a versioned JSON schema before accepting it.
8. THE Grimoire SHALL reject any Import_File whose size exceeds 5 MB and SHALL display a plain-language message to the user.
9. THE Grimoire SHALL render all imported text content (spellbook names, card names, object labels, spell text) as plain text, never as HTML or executable code.
10. WHEN an Import_File is validated, THE Grimoire SHALL re-parse every spell text found in the file; IF any spell fails to parse, THEN THE Grimoire SHALL reject the entire Import_File and display a plain-language error message identifying which spell failed and on which card.
11. Export_Files SHALL include stable local object identifiers; WHEN an Import_File is imported, THE Grimoire SHALL remap all object identifiers to new unique values to prevent collisions with existing spellbooks.
12. WHEN validating an Import_File, THE Grimoire SHALL verify that every spell's target object identifier references an existing object on the same card; IF any spell references a non-existent object, THE Grimoire SHALL reject the Import_File and display a plain-language error message.
13. WHEN validating an Import_File, THE Grimoire SHALL verify that all card names are unique within the spellbook (case-insensitive) and that all field identifiers are unique within each card; IF any duplicates are found, THE Grimoire SHALL reject the Import_File and display a plain-language error message.

---

### Requirement 8: Example Spellbooks [Core]

**User Story:** As a new user, I want to explore pre-built example spellbooks, so that I can learn how Grimoire works without building from scratch.

#### Acceptance Criteria

1. THE Grimoire SHALL include three built-in example spellbooks: a quiz, a shop price list, and a flashcard deck.
2. THE Grimoire SHALL make each example spellbook available from the application's start screen.
3. WHEN the user opens an example spellbook, THE Grimoire SHALL load it into a working copy that the user can edit and save without altering the original example.
4. THE Grimoire SHALL ensure each example spellbook contains at least one spell that exercises navigation, field reading or setting, and showing a message.
5. THE Grimoire SHALL not use Apple names, logos, assets, or sample content in any example spellbook.
6. WHEN Grimoire starts, THE Grimoire SHALL verify that all three example spellbooks parse and load without errors; IF any example fails to parse, THEN THE Grimoire SHALL log a console error but SHALL NOT prevent the application from starting.

---

### Requirement 9: Visual Style and Accessibility [Core]

**User Story:** As a user, I want Grimoire to look distinctive and to be fully usable with a keyboard and screen reader, so that the tool is accessible regardless of my input method or motion sensitivity.

#### Acceptance Criteria

1. THE Grimoire SHALL apply a spooky-retro visual style with high-contrast, candlelit color palette across all UI surfaces.
2. THE Grimoire SHALL provide a Reduced_Motion_Mode that disables non-essential animations and transitions.
3. WHEN the operating system reports `prefers-reduced-motion: reduce`, THE Grimoire SHALL activate Reduced_Motion_Mode automatically.
4. THE Grimoire SHALL ensure all interactive controls are reachable and operable by keyboard alone, following logical tab order.
5. THE Grimoire SHALL ensure all text and icon controls meet WCAG 2.1 AA contrast ratio requirements (minimum 4.5:1 for normal text, 3:1 for large text and UI components).
6. THE Grimoire SHALL provide visible focus indicators on all interactive elements while in keyboard navigation.
7. THE Grimoire SHALL provide ARIA labels or visible text labels on all interactive controls so that screen readers can announce their purpose.
8. IF a spooky visual effect obscures content or controls, THEN THE Grimoire SHALL remove or suppress that effect.

---

### Requirement 10: Storage Failure Handling [Core]

**User Story:** As a user, I want Grimoire to tell me when it cannot save my work, so that I do not lose changes without warning.

#### Acceptance Criteria

1. WHEN a Storage write operation fails (e.g. quota exceeded or storage unavailable), THE Grimoire SHALL display a plain-language warning to the user immediately.
2. THE Grimoire SHALL not silently discard unsaved changes when a Storage write operation fails.
3. THE Grimoire SHALL continue to allow the user to export the current spellbook as a file even when Storage is unavailable.

---

### Requirement 11: Start Screen [Core]

**User Story:** As a user, I want a start screen where I can see, open, rename, and delete my spellbooks, so that I can manage my work in one place.

#### Acceptance Criteria

1. THE Grimoire SHALL display a start screen that lists all spellbooks saved in Storage alongside all built-in example spellbooks.
2. THE Grimoire SHALL visually distinguish user spellbooks from example spellbooks on the start screen.
3. WHEN the user selects a spellbook on the start screen, THE Grimoire SHALL open that spellbook and display its first card.
4. THE Grimoire SHALL allow the user to create a new spellbook from the start screen.
5. THE Grimoire SHALL allow the user to rename any user spellbook from the start screen.
6. WHEN the user requests deletion of a spellbook from the start screen, THE Grimoire SHALL request confirmation before removing it from Storage.
