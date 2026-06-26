# Transactional Overlay Shell DSL (.ash)

**Designed by Tanmoy Maji (24CS10037)**

---

# Overview

This DSL is a **transactional shell-like scripting language** designed for safe filesystem manipulation. Instead of directly modifying the filesystem like traditional shells, commands run inside **overlay layers**. Changes can be committed, merged, or discarded safely.

The language combines ideas from:

* Traditional Unix shells
* Transactional systems
* Layered filesystems
* Structured programming languages

Primary goals of the language:

* Safe filesystem experimentation
* Strong typing for filesystem objects
* Structured scripting syntax
* Controlled integration with Linux commands
* Transaction-like filesystem behavior

All source files use the extension:

```
.ash
```

Example:

```
build.ash
backup.ash
monitor.ash
```

The DSL is designed as an **interpreter-based language**, not a wrapper over Bash. Programs are parsed, executed through a runtime environment, and interact with the filesystem through controlled overlay layers.

---

# Core Concept: Overlay-Based Execution

## Default Execution Model

Every script runs inside **at least one overlay layer** unless native mode is enabled.

```
base filesystem
        ↓
workspace overlay
```

All filesystem operations modify the **overlay layer instead of the real filesystem**.

This ensures:

* accidental destructive commands cannot affect the real filesystem
* scripts can be tested safely
* partial failures do not corrupt the real system

If the overlay layer is destroyed, all temporary filesystem modifications disappear.

---

# Overlay Control

## overlay {}

Creates a temporary overlay layer.

```ash
overlay {
    file f = create("temp.txt")
}
```

Execution behavior:

1. A new overlay layer is created.
2. The block executes inside that overlay.
3. When the block ends:

   * the overlay is destroyed unless changes are committed.

This provides a **transaction-like sandbox**.

---

# Global Native Execution

Automatic overlays can be disabled globally.

```
#run all native
```

Example:

```ash
#run all native

file f = create("real_file.txt")
```

Now operations execute directly on the real filesystem.

However overlays can still be created manually:

```ash
overlay {
    file safe = create("safe.txt")
}
```

---

# Native Execution

A function can run **without creating a new overlay**.

```ash
native fn cleanup() {
    delete("*.tmp")
}
```

Native execution means:

```
current layer
   ↓
function executes directly
```

No additional overlay layer is created.

---

# Persisted Function Execution

A function can also automatically **merge its overlay into the parent layer**.

```ash
persist fn build() -> file {
    file f = compile()
    return f
}
```

Behavior:

1. Function creates an overlay
2. Execution occurs inside the overlay
3. On successful completion the overlay is **merged into the parent layer**

This behaves like a **transaction commit**.

---

# Call-Level Execution Modifiers

Function calls can override the default behavior.

Example:

```ash
build().persist()
cleanup().native()
```

Modifiers:

```
.native()   → execute without creating overlay
.persist()  → merge overlay into parent layer
```

Call modifiers override function defaults.

---

# Data Types

Primitive numeric and text types:

```
int
float
string
bool
```

Filesystem types:

```
file
folder
```

Command stream type:

```
stream
```

List types exist for **all types** using `[]`.

Examples:

```ash
int count = 10
float ratio = 3.14
string name = "logs"
bool found = true

file log = "system.log"
folder dir = "data"
```

List examples:

```ash
int[] nums = [1,2,3]
string[] names = ["a","b","c"]
file[] logs = ["a.log","b.log"]
```

---

# Variables

Variables follow standard structured language rules.

Properties:

* block scoped
* function scoped
* loop scoped

Examples:

```ash
int x = 5
string name = "test"
bool ready = true
```

Filesystem objects reference files inside overlay layers.

---

# List Operations

Lists exist for all types.

Example:

```ash
file[] logs = ["a.log","b.log"]
```

Lists support indexing:

```ash
file first = logs[0]
```

Built-in functions:

```
len(list)
append(list,value)
pop(list)
```

Example:

```ash
int n = len(logs)
```

---

# File Persistence

Files created inside overlays disappear unless explicitly persisted.

## persist(depth = N)

```ash
file f = create("data.txt")
f.persist(depth = 2)
```

Meaning:

The file survives **two overlay exits**.

Example stack:

```
layer3
layer2
layer1
base
```

If depth = 2:

```
file survives layer3 and layer2
```

---

## Global Persistence

```
f.persist(depth = INF)
```

Meaning:

The file persists all the way to the **base filesystem**.

---

# Arithmetic Operators

Supported operators:

```
+  -  *  /
```

Examples:

```ash
int x = 5 + 3
float y = 10 / 4
```

Operator precedence:

1. `* /`
2. `+ -`

Division rules:

```
int / int → float
```

---

# File Collection Operators

Filesystem objects support `+` to create lists.

Rules:

```
file + file → file[]
file[] + file → file[]
file[] + file[] → file[]
```

Same rules apply to folders.

Examples:

```ash
file a = "a.txt"
file b = "b.txt"

file[] files = a + b
```

```ash
file[] logs = ["a.log"]
file err = "error.log"

file[] all = logs + err
```

---

# Comparison Operators

Supported comparison operators:

```
==  equality
!=  inequality
<   less than
>   greater than
<=  less or equal
>=  greater or equal
```

Example:

```ash
if count > 10 {
    print("large")
}
```

---

# Logical Operators

Logical expressions support:

```
&&   AND
||   OR
!    NOT
```

Example:

```ash
if count > 10 && exists(log) {
    process(log)
}
```

---

# Conditional Statements

Example:

```ash
if exists(log) {
    process(log)
}
else {
    print("missing log")
}
```

Conditions must evaluate to `bool`.

---

# Function System

## Function Definition

```ash
fn build() {
    file out = compile()
}
```

Functions normally create a new overlay layer.

---

## Function Return Behavior

Returned filesystem objects automatically persist.

Rule:

```
return file → persist(depth ≥ 1)
```

Example:

```ash
fn build() -> file {
    file f = compile()
    return f
}
```

Returned files survive the function overlay.

---

# External Command Execution

Commands are executed using `run`.

Example:

```ash
stream s = run grep("error", log)
```

Arguments may be:

* files
* strings
* streams

---

# Streams

`stream` represents the combined output of a command.

It contains:

```
stdout
stderr
exit code
```

Example:

```ash
stream s = run grep("error", log)
```

Streams can be converted:

```ash
string text = string(s)
int count = int(run wc("-l", s))
```

---

# Pipelines

Pipelines use nested function calls instead of pipe syntax.

Traditional shell:

```
cat log | grep error | wc -l
```

DSL equivalent:

```ash
stream result = run wc("-l",
    run grep("error",
        run cat(log)))
```

---

# Loops

## while Loop

```ash
while condition {
    ...
}
```

Execution model:

```
loop overlay
     ↓
iteration overlay
```

After each iteration:

```
iteration overlay → merged into loop overlay
```

---

## for Loop

```ash
for f in files {
    process(f)
}
```

Works with all list types.

---

# sleep

Pauses execution.

```ash
sleep(2s)
```

Supported units:

```
ms
s
m
```

Example:

```ash
while true {
    print("heartbeat")
    sleep(1s)
}
```

---

# TTL Block

Limits execution time of a code section.

```
ttl (15s) {
    ...
}
```

Behavior:

* code runs for maximum specified time
* execution stops when time limit is reached
* control returns to outer scope

Important rule:

```
ttl does NOT create an additional overlay
```

---

# Error Handling

## try / catch

```ash
try {
    delete(file)
}
catch e {
    print("error")
}
```

Execution model:

```
parent layer
      ↓
try overlay
```

If an error occurs:

```
try overlay discarded
catch executed
```

---

## finally

Optional cleanup.

```ash
try {
    run command()
}
catch e {
    print("error")
}
finally {
    cleanup()
}
```

---

# Keyboard Control

## Ctrl + B

Loop break mechanism.

Behavior:

* first press → prepare break
* second press → break most recent loop

Current iteration overlay is discarded.

---

## Ctrl + C

Program interruption behavior.

```
first press → interrupt running command
second press → terminate program
```

---

# strace Integration

System call tracing is supported.

Example:

```ash
strace {
    run grep("error", log)
}
```

Displays system calls performed by the command.

Useful for debugging filesystem operations.

---

# Comments

## Single-line

```ash
// comment
```

## Multi-line

```ash
/*
multi line
comment
*/
```

---

# File Inclusion

External files can be included.

```ash
#include "utils.ash"
```

Contents are inserted during preprocessing.

---

# Example Program

```ash
file log = "system.log"

while true {

    try {

        stream s = run grep("error", log)

        int count = int(run wc("-l", s))

        print(count)

    }

    catch e {

        print("command failed")

    }

    sleep(5s)

}
```

---

# Key Advantages

1. Filesystem safety via overlays
2. Transaction-like scripting
3. Strong typing for filesystem objects
4. Structured programming constructs
5. Safe experimentation environment
6. Native Linux command integration
7. Deterministic execution model

---

# Summary

The `.ash` DSL provides a **safe, structured, and powerful scripting environment** for filesystem manipulation.

It combines:

* the flexibility of Unix shells
* the safety of transactional overlays
* the clarity of structured programming

This allows developers to experiment with filesystem operations without risking destructive system changes.
