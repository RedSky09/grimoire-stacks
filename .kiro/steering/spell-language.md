# Spell Language Reference

This document provides a complete reference for the Grimoire Spell Language, including grammar, reserved keywords, escape sequences, and comprehensive examples.

---

## Overview

The Spell Language is a small, English-like scripting language that enables non-programmers to add interactive behavior to Grimoire spellbooks. All spell text must parse to an abstract syntax tree (AST) before execution—no raw code evaluation is performed.

**Core Principles:**
- **Safety-first**: All spell text parses to an AST before execution
- **Sandboxed**: Spells run in a restricted interpreter with no DOM/network/storage access
- **Deterministic**: Same inputs produce same outputs
- **Case-insensitive keywords**: Keywords like `go`, `to`, `card` are case-insensitive
- **Text-based values**: All values (including numbers) are compared as text strings

---

## Complete Grammar (EBNF)

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

---

## Reserved Keywords

The following keywords are **reserved** and cannot be used as Field_Identifiers (case-insensitive):

- `go`
- `to`
- `card`
- `set`
- `show`
- `message`
- `if`
- `then`
- `else`
- `end`
- `field`
- `is`
- `not`
- `contains`

**Examples:**
- ✅ Valid Field_ID: `username`, `answer_text`, `score1`
- ❌ Invalid Field_ID: `field` (reserved keyword)
- ❌ Invalid Field_ID: `show` (reserved keyword)
- ❌ Invalid Field_ID: `123user` (starts with digit)

---

## String Escape Sequences

String literals support the following escape sequences:

| Escape Sequence | Meaning |
|----------------|---------|
| `\"` | Literal double quote character |
| `\\` | Literal backslash character |
| `\n` | Newline character |

**Examples:**
```
show message "Hello \"World\""
```
Displays: `Hello "World"`

```
show message "Line 1\nLine 2"
```
Displays:
```
Line 1
Line 2
```

```
show message "Path: C:\\Users\\Documents"
```
Displays: `Path: C:\Users\Documents`

---

## Statement Types

### 1. Go To Card

Navigates to a specified card by name or number and **terminates spell execution**. Any statements after `go to card` are unreachable.

**Syntax:**
```
go to card <card_target>
```

**Card Target Options:**
- **Card Name** (string literal): `go to card "Results"`
- **Card Number** (integer): `go to card 2`

**Behavior:**
- Card names are matched **case-insensitively** at runtime
- If the target card has a `card-open` spell, it executes automatically
- Navigation depth is limited to 16 to prevent infinite loops

**Examples:**
```
go to card "Welcome Screen"
```

```
go to card 1
```

```
go to card "Quiz Results"
```

---

### 2. Set Field

Writes a value to a text field on the current card.

**Syntax:**
```
set <field_id> to <expression>
```

**Rules:**
- Field identifiers are **scoped to the current card**
- Field reads in the same spell reflect earlier set operations
- Field identifiers must exist on the current card

**Examples:**

Set a field to a string literal:
```
set username to "Alice"
```

Set a field to a number:
```
set score to 100
```

Copy one field to another:
```
set answer_backup to field answer
```

Set a field based on a calculation (using string concatenation):
```
set result to "Score: 95"
```

---

### 3. Show Message

Displays a modal message to the user.

**Syntax:**
```
show message <expression>
```

**Examples:**

Show a simple message:
```
show message "Hello, World!"
```

Show a field value:
```
show message field username
```

Show a message with escape sequences:
```
show message "Welcome!\nPlease enter your name."
```

Show a multi-line formatted message:
```
show message "Quiz Complete\nYour score: 8/10"
```

---

### 4. If-Then-Else

Executes statements conditionally based on a boolean condition.

**Syntax:**
```
if <condition> then
  <statement>
  ...
else
  <statement>
  ...
end if
```

**Rules:**
- The `else` block is optional
- The `end if` is **required**
- Statements in branches must be on separate lines
- Newlines are significant (they separate statements)

**Condition Operators:**
- `is` — equality test (case-sensitive for strings)
- `is not` — inequality test (case-sensitive)
- `contains` — substring test (case-sensitive)

**Examples:**

Simple if-then:
```
if field answer is "correct" then
  show message "Well done!"
end if
```

If-then-else:
```
if field answer is "correct" then
  show message "Well done!"
else
  show message "Try again"
end if
```

Using `is not`:
```
if field username is not "" then
  show message "Hello"
else
  show message "Please enter your name"
end if
```

Using `contains`:
```
if field response contains "yes" then
  show message "Affirmative response detected"
end if
```

Nested conditions:
```
if field score is "100" then
  show message "Perfect score!"
else
  if field score contains "9" then
    show message "Great job!"
  else
    show message "Keep practicing"
  end if
end if
```

Complex example with field manipulation:
```
if field answer is "blue" then
  set result to "Correct!"
  set score to 10
  show message "You got it right!"
else
  set result to "Incorrect"
  set score to 0
  show message "Try again"
end if
```

---

## Expressions

Expressions produce values that can be used in conditions or statements.

### String Literal
A quoted string with escape sequences.

**Syntax:** `"text content"`

**Examples:**
```
"Hello, World!"
"Line 1\nLine 2"
"Quote: \"To be or not to be\""
```

### Number Literal
An integer number (stored as text internally).

**Syntax:** One or more digits

**Examples:**
```
42
100
0
999
```

**Note:** Numbers are compared as text strings, so `"100"` is treated the same as `100`.

### Field Read
Reads the current runtime value of a text field.

**Syntax:** `field <field_id>`

**Examples:**
```
field username
field score
field answer_text
```

**Important:** Field reads reflect any earlier `set` operations in the same spell execution.

---

## Conditions

Conditions evaluate to true or false and are used in `if` statements.

### Equality: `is`
Tests if two expressions are equal (case-sensitive string comparison).

**Example:**
```
if field username is "Alice" then
  show message "Welcome back, Alice!"
end if
```

### Inequality: `is not`
Tests if two expressions are not equal (case-sensitive).

**Example:**
```
if field password is not "" then
  show message "Password entered"
end if
```

### Contains
Tests if the left-hand string contains the right-hand string as a substring (case-sensitive).

**Example:**
```
if field email contains "@" then
  show message "Valid email format"
end if
```

---

## Complete Examples

### Example 1: Simple Button Navigation
**Context:** Button click event that navigates to the next card.

```
go to card 2
```

### Example 2: Field Validation with Message
**Context:** Button click that checks if a username field is filled.

```
if field username is "" then
  show message "Please enter a username"
else
  show message "Hello, friend!"
end if
```

### Example 3: Quiz Answer Check
**Context:** Button click that checks an answer and navigates based on correctness.

```
if field answer is "Paris" then
  set result to "Correct!"
  show message "That's right!"
  go to card "Congratulations"
else
  set result to "Incorrect"
  show message "Try again"
end if
```

### Example 4: Card-Open Event
**Context:** Spell attached to card-open event that initializes fields.

```
set welcome_msg to "Welcome to the quiz!"
show message field welcome_msg
```

### Example 5: Multi-Step Form Validation
**Context:** Button click that validates multiple fields.

```
if field name is "" then
  show message "Please enter your name"
else
  if field email is "" then
    show message "Please enter your email"
  else
    if field email contains "@" then
      set form_status to "Complete"
      show message "Form submitted successfully"
      go to card "Thank You"
    else
      show message "Please enter a valid email"
    end if
  end if
end if
```

### Example 6: Score Calculation
**Context:** Button click that updates a score based on a correct answer.

```
if field question1 is "correct" then
  set score to "10"
else
  set score to "0"
end if
show message field score
```

### Example 7: Flashcard Navigation
**Context:** "Next" button on a flashcard that cycles through cards.

```
go to card 2
```

### Example 8: Conditional Navigation
**Context:** Button that navigates to different cards based on user input.

```
if field difficulty is "easy" then
  go to card "Easy Quiz"
else
  if field difficulty is "hard" then
    go to card "Hard Quiz"
  else
    go to card "Medium Quiz"
  end if
end if
```

---

## Runtime Behavior Notes

### Field Scope
- Field identifiers are **scoped to the current card only**
- A spell cannot read or write fields on other cards
- Referencing a non-existent field causes a runtime error

### Navigation and Card-Open Events
- When `go to card` executes, the target card's `card-open` spell (if any) runs automatically
- Navigation depth is limited to 16 to prevent infinite loops
- After 16 chained navigations, execution halts with an error

### Step Limit
- Each spell execution is limited to 10,000 steps
- Each expression evaluation, condition check, and statement execution counts as a step
- If the limit is reached, execution halts with: "This spell was stopped because it performed too many steps."

### Determinism
- Given the same inputs (AST, card state, field values), spell execution produces the same outputs
- No randomness, no external state, no side effects

### Case Sensitivity
- **Keywords** are case-insensitive (`GO`, `go`, `Go` are all valid)
- **Field identifiers** are case-sensitive (`username` ≠ `Username`)
- **String comparisons** are case-sensitive (`"hello"` ≠ `"Hello"`)
- **Card names** are matched case-insensitively (`"Welcome"` matches `"welcome"`)

---

## Parser Notes

### Tokenization
- Keywords, identifiers, string literals, and numbers are tokenized
- Newlines are significant in `if` blocks (they separate statements)
- Whitespace between tokens is ignored

### Error Reporting
- Parser errors include line and column numbers
- Error messages are plain-language (e.g., "Expected 'to' on line 1 but found 'too'")

### Round-Trip Property
- Parsing spell text, converting the AST back to text, and parsing again produces an equivalent AST
- This ensures the parser and pretty-printer are consistent

---

## Common Patterns

### Pattern: Input Validation
```
if field email is "" then
  show message "Email is required"
else
  if field email contains "@" then
    set validation_status to "valid"
  else
    show message "Invalid email format"
  end if
end if
```

### Pattern: Conditional Navigation
```
if field score is "100" then
  go to card "Perfect Score"
else
  go to card "Try Again"
end if
```

### Pattern: Field Copying
```
set backup_answer to field user_answer
set result to "Answer saved"
show message field result
```

### Pattern: Multi-Step Workflow
```
if field step is "1" then
  set step to "2"
  show message "Moving to step 2"
else
  if field step is "2" then
    set step to "3"
    show message "Moving to step 3"
  else
    show message "Workflow complete"
    go to card "Results"
  end if
end if
```

---

## Implementation Guidelines for AI Agents

When working with the Spell Language:

1. **Always validate syntax** before execution—parse to an AST first
2. **Check field references** against the current card's field list
3. **Respect field scope**—fields are card-local only
4. **Handle navigation carefully**—track navigation depth to prevent infinite loops
5. **Count steps**—enforce the 10,000 step limit
6. **Preserve determinism**—no side effects, no external state
7. **Report errors clearly**—use plain language with line numbers
8. **Test round-trip parsing**—ensure parse → print → parse equivalence

---

## Validation Requirements

When parsing or generating spell text:

- **Field_ID format**: Must match `/^[a-zA-Z_][a-zA-Z0-9_]*$/` and not be a reserved keyword
- **Card names**: Must be quoted string literals when used as targets
- **String literals**: Must be double-quoted with proper escape sequences
- **If blocks**: Must have `then`, newlines for statements, and `end if`
- **Go to card**: Terminates execution; warn about unreachable statements

---

## Related Requirements

This steering file documents functionality defined in:
- **Requirement 4.1**: Statement types (go to card, set field, show message, if-then-else)
- **Requirement 4.2**: Reserved keywords
- **Requirement 4.3**: String escape sequences
- **Requirement 4.9**: Text-based value comparisons
- **Requirement 4.11**: Round-trip parsing property
- **Requirement 5.2**: Step limit enforcement
- **Requirement 5.6**: Determinism property
- **Requirement 5.7**: Navigation depth limit
