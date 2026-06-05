---
name: st-twincat
description: Structured Text according to IEC 61131-3 with Beckhoff TwinCAT 3 conventions — covering naming, architecture, lifecycle, error handling, determinism, and strict review rules.
---

# Structured Text — IEC 61131-3 / Beckhoff TwinCAT 3

You are working with Structured Text (ST) targeting Beckhoff TwinCAT 3 PLC under IEC 61131-3.

Follow these rules strictly unless the user explicitly provides different project rules.

## Scope

Apply this skill when:
- writing, reviewing, or refactoring ST code
- designing TwinCAT PLC architecture
- working with function blocks, methods, interfaces, structs, enums, unions, actions, and properties
- discussing naming, lifecycle, state machines, diagnostics, timing, motion, fieldbus, MQTT, ADS, or real-time behavior
- converting designs from OOP languages into IEC 61131-3 / TwinCAT style

## Core principles

- Target platform is Beckhoff TwinCAT 3 PLC.
- Language is Structured Text under IEC 61131-3 including OOP extensions.
- Beckhoff-specific behavior and common TwinCAT idioms apply.
- Keyword capitalization is not semantic in ST, but generated code must be stylistically consistent throughout a project. 
- Code must be deterministic, explicit, and scan-cycle-safe.
- Code must be easy to debug online.
- Prefer composition over inheritance. Avoid inheritance whenever possible. If inheritance is used, limit it to one level.
- All allocation is static. No dynamic memory allocation.



## Naming

### General

- No prefixes or postfixes on any name, except `I` for interfaces.
- Names describe intent, not type or implementation detail.
- Names must be clear and unambiguous without requiring context.
- Methods that having a helper character which are often proteced or private method should have a "_" as prefix. 
- Abstract classes can't be reviewed like a full implementation. No used variables have to be used in the child classes

### Capitalization

| Element                        | Style       | Example              |
|-------------------------------|-------------|----------------------|
| Variables                      | camelCase   | `speedActual`        |
| Methods                        | camelCase   | `enable()`           |
| Function blocks                | PascalCase  | `AxisController`     |
| Types (struct, enum, union)    | PascalCase  | `ControllerState`    |
| Properties                     | PascalCase  | `IsReady`            |
| Interfaces                     | I + PascalCase | `IControllable`   |
| Constants                      | UPPERCASE   | `MAX_AXES`           |
| Enum values                    | UPPERCASE   | `IDLE`, `RUNNING`    |


- Capitalization is for readability and without function, since IEC61131-3 doesn't care about capitalization.

### Internal variables

- Internal variables use a leading underscore when there is a name collision with properties or methods.
- Internal variables are never exposed directly. Use properties for external access.

### Word order

- Subject first, modifier second.

Good: `speedMax`, `torqueAverage`, `temperatureFiltered`, `pressureSetpoint`, `lengthMin`

Bad: `maxSpeed`, `averageTorque`, `filteredTemperature`

### Consistency

- The same physical quantity or concept uses the same subject name across the entire system.
- If one module calls it `speed`, every module calls it `speed` — never `velocity` in one place and `speed` in another.

### Methods and properties

- Methods represent actions: `enable()`, `reset()`, `init()`.
- Properties represent states: `IsEnabled`, `HasError`, `Position`.



## Types

### Preferred types

- Use `DINT` for integers.
- Use `LREAL` for floating point.
- Use `STRING` or `STRING(n)` for text.
- Use `BOOL` for flags.

Use `INT`, `UINT`, `REAL`, `BYTE`, `WORD`, `DWORD`, `LWORD`, `SINT`, `USINT`, `UDINT`, `ULINT`, `LINT` only when the hardware interface, protocol, or memory layout requires it.

### STRING safety

- Always specify explicit length: `STRING(255)`, never bare `STRING` for buffers that receive external data.
- Be aware that `STRING` defaults to 80 characters in TwinCAT.
- STRING comparison is case-sensitive in TwinCAT. Use `F_ToUpper` or `F_ToLower` from `Tc2_Utilities` when case-insensitive comparison is needed.
- Avoid `WSTRING` unless the application explicitly requires Unicode.

### Implicit conversions

- Avoid relying on implicit type conversions. Use explicit conversion functions (`TO_LREAL`, `TO_DINT`, etc.).
- Be aware that `REAL` to `LREAL` promotion can introduce precision artifacts. Convert early and stay in `LREAL`.


## Constants and enums

- Constants are `UPPERCASE`.
- Enum values are `UPPERCASE`.
- Enum values must not repeat the type name.
- Use typed enums with explicit base type.

Good:
```
TYPE ControllerState :
(
    IDLE,
    RUNNING,
    STOPPING,
    ERROR
) DINT;
END_TYPE
```

Bad:
```
TYPE ControllerState :
(
    CONTROLLER_STATE_IDLE,
    CONTROLLER_STATE_RUNNING
);
END_TYPE
```
## Classes

- An abstract class can have empty methods and properties
- A child class has the `EXTENDS` keyword in the declaration header
- Child classes inherit all methods and properties of the parent class

## Arrays

- Always 0-based indexing: `ARRAY[0..N-1]`.
- Never use 1-based arrays.
- Define array bounds with named constants: `ARRAY[0..MAX_AXES - 1] OF AxisData`.

## Loops

- Loop index variables use single uppercase letters: `N`, `M`, `K`, `Q`.
- Never use `i`, `j`, `k` or any lowercase single-letter variable.
- Loop index variables are used only as indices — never reused for other purposes.
- Avoid unbounded or long-running loops. Every loop must have a known maximum iteration count at compile time.

## IO structures

- An none trivial IO-struct consists usually of 3 parts
	1. <CLASS>IO 
    2. <CLASS>_In 
    3. <CLASS>_Out

- Trivial IO-structs are can have 1 to 2 files

## Sequence of Methods and Properties

Twincat sorts methods and properties automatically

## Architecture

### Composition over inheritance

- Build behavior by composing function blocks inside other function blocks.
- Do not use inheritance to share behavior. Use interfaces to define contracts and composition to provide implementation.
- If inheritance is unavoidable, limit to one level. Document why.

### Interface-based design

- Define behavioral contracts as interfaces (`I` prefix).
- Classes implementing the same interface do not have necessarily similar implementations!
- Function blocks implement interfaces explicitly.
- Depend on interfaces, not concrete function blocks, when a component needs to interact with interchangeable implementations.
- Keep interfaces small and focused (interface segregation).
- All variables including interface arrays are initialized to 0 or the given initialization value. 

### Guard clauses for `REFERENCE TO` and `INTERFACE`

- Guard clauses are not used in helper functions
- references declared by  `REFERENCE TO` have to be tested with `__ISVALIDREF()`
- referenced declared by `INTERFACE` have to be tested on 0 . Example: `IF service <> 0 THEN ...`
- If and guard clause for reference contains more than one condition, use keyword AND_THEN and test the reference first

### Static allocation

- All instances are statically allocated in `VAR` sections.
- Variables for functionsblocks or classes in methods have to be instanced in a `VAR_INST` block or on function block level. 
- No use of `__NEW`, `__DELETE`, or dynamic memory.
- No `POINTER TO` for ownership. Use `POINTER TO` only for hardware-mapped access or library interop where unavoidable.

### REFERENCE TO vs POINTER TO

- Prefer `REFERENCE TO` over `POINTER TO` for passing references between function blocks.
-  A reference instanciated with `REFERENCE TO` can be checked with __ISVALIDREF() 
- `POINTER TO` is acceptable only for usage of libraries provided by third parties  and low-level library interop.
- Never store a `POINTER TO` a local variable beyond the scope of the call.

### VAR sections

- `VAR_INPUT`: data flowing into the block. Caller sets, block reads.
- `VAR_OUTPUT`: `REFERENCE TO` replaces this section
- `VAR`: internal state. Not visible externally.
- `VAR_INST`: static variables for functions and methods.
- Never use `VAR_IN_OUT`.  `REFERENCE TO` replaces this section
- Never use `VAR_OUTPUT`.  `REFERENCE TO` and `Property` replaces this section 

### Math and units
- Use small epsilon thresholds for comparisons (`ABS(x) > 1E-6`).

## Lifecycle

### FB_init and FB_exit

- Use an explicit `init()` or `init_base()` method called from the application for controlled initialization with parameters.
- `FB_init` is used only in special cases and is called automatically on initialization and online change. 
- Do not put complex logic in `FB_init`. It runs outside the normal scan cycle and has no guaranteed execution order relative to other blocks.
- `FB_exit` is called on shutdown and before online change. Use it to release external resources (file handles, ADS connections) if applicable.
- Be aware that `FB_init` is called again after every online change. Code in `FB_init` must be idempotent.

### Initialization pattern

- Every function block that requires setup exposes an `init()` method.
- `init()` is called explicitly from the application after all instances are created at the time of an explicit initialization phase. 
- Initialization order is controlled by the application, not by implicit constructor ordering.
- Use a `_initialized` flag to guard against double-init and to allow cyclic code to detect uninitialized blocks.

### Online change safety

- Never rely on variable initialization values surviving online change unless the variable is `PERSISTENT`.
- Test that code behaves correctly after an online change — `FB_init` re-runs, `VAR` values may reset.
- Use `{attribute 'no_assign'}` on function blocks that must not be copied, to prevent accidental assignment.

## State machines

- Use enum-based state with a `CASE` structure.
- Every state machine has an explicit enum type for its states.
- Transitions are explicit — each state defines exactly which states it can transition to and under what conditions.
- Every `CASE` has an `ELSE` branch that handles unexpected states (transition to `ERROR` or `IDLE`).
- Avoid nested state machines when possible. Prefer flat state machines composed with sub-blocks.
- Use an `update()` method to update  state machines

## Timers and timing

- Use `TON`, `TOF`, `TP` from `Tc2_Standard`.
- Timer instances must be called every scan cycle — never conditionally skip a timer call.
- Declare timer instances in `VAR` on function block level, and `VAR_INST` on method level.
- Do not compare `ET` (elapsed time) with `=`. Use `>=` for threshold checks because `ET` increments in scan-cycle steps and may skip the exact value.
- When a timeout represents a parameter, define it as a `TIME` constant or `VAR_INPUT`, not a magic literal.

## SEL()

- The SEL() operator is specified as SEL(condition, Expression for False, Expression for True) -> Expression value
- The SEL() operator is defined according to IEC61131-3. 

```
result := SEL( x > 0, ResultFalse, ResultTrue);
```

## Error and State handling

- Use a consistent error pattern across all function blocks:
  - `OpState` property indicates the operational state. In simpel cases it's just OP and NOOP.
  - `ExecState` property indicates an exec the state of an instance with IDLE, PREPARE, ACTIVE and STOPPING 
- Provide a `reset()` method to clear errors and return to a known state.
- Errors propagate upward through composition — a parent block reflects child errors.
- Never silently swallow errors. Every error must be observable from the outside.

## Pragmas and attributes

- `{attribute 'qualified_only'}` — apply to enums so values must be accessed as `EnumType.VALUE`. Prefer this for all enums to avoid name collisions.
- `{attribute 'strict'}` — apply to enums to enable strict type checking. Combine with `qualified_only`.
- `{attribute 'no_explicit_call' := 'do not call this POU directly'}` - apply to classes  or functionblocks 
- `{attribute 'conditionalshow_all_locals'}` apply to classes or functionblocks

## Persistent and retain variables

- `RETAIN` variables survive a warm restart but not a cold restart or download. Don't use RETAIN
- `PERSISTENT` variables survive download and cold restart (stored in persistent file).
- Use `PERSISTENT` for recipe data, calibration values, and counters that must survive power cycles.
- Minimize the number of `PERSISTENT` and `RETAIN` variables — they have limited storage and impact boot time.
- Never put large arrays or complex structures in `PERSISTENT` without understanding the storage impact.

## Scan cycle and real-time

- Every function block's cyclic method must complete within a single scan cycle. No blocking, no waiting, no loops that span multiple cycles.
- Long-running operations must be implemented as state machines that do incremental work each cycle.
- Avoid `WHILE` loops with exit conditions that depend on external state — they can block the scan cycle.
- Be aware of task cycle times. Code in a 1 ms task has a much tighter time budget than code in a 10 ms task.
- Separate fast I/O processing from slow logic (HMI, logging, communication) into different tasks with appropriate priorities.

## Separation of concerns

- `MAIN` program is the entry point. It calls top-level function blocks in a defined order.
- `MAIN` contains no business logic — only instantiation and cyclic calls.
- Initialization sequences are handled by a dedicated startup state machine or init coordinator, not scattered across `MAIN`.
- I/O mapping is separated from logic. Map physical I/O to a dedicated I/O function block or GVL, then pass values into logic blocks via inputs.
- HMI data exchange is separated from control logic. Use dedicated HMI interface structs or GVLs.

## File format and structure
- PLC sources are stored as XML (`.TcPOU`, `.TcDUT`, `.TcIO`).
- Do not change XML headers/encodings; keep the BOM/encoding intact.
- Keep the `<Declaration>` and `<Implementation>` CDATA blocks intact.
- Use `FUNCTION_BLOCK`, `METHOD`, and `PROPERTY` consistently.

## Libraries

- `Tc2_Standard` — standard IEC function blocks (timers, triggers, counters). Always available.
- `Tc2_System` — system functions (memory operations, task info, ADS). Use for system time and  task cycle time queries.
- `Tc2_Utilities` — string utilities, conversions, file operations. Use for `F_ToUpper`, `F_ToLower`, `CONCAT2`, etc.
- `Tc3_Module` — TcCOM module development. Use only when building TcCOM objects.
- Do not add library references that are not used. Keep dependencies minimal.

## Review rules

### Reject code that has

- Wrong word order in names (modifier before subject).
- Hungarian notation or type prefixes/postfixes (except `I` for interfaces).
- Lowercase constants or enum values.
- 1-based arrays.
- Loop variables named `i`, `j`, `k` or any lowercase single letter.
- Inconsistent naming of the same concept across modules.
- Dynamic memory allocation (`__NEW`, `__DELETE`).
- Bare `POINTER TO` without null check before dereference.
- Timer instances that are not called every cycle.
- `FB_init` with complex logic or external dependencies.
- Silent error swallowing (error occurs but no external indication).
- Magic numbers without named constants.
- Implicit type conversions where precision loss is possible.
- Missing `ELSE` branch in state machine `CASE` blocks.

### Ensure code has

- Deterministic, scan-cycle-safe logic.
- Clear separation of state (properties) and actions (methods).
- Explicit initialization lifecycle with `init()` method.
- Consistent error handling pattern (`OpState`, `ExecState`, `reset()`).
- TwinCAT-compatible semantics verified against known platform behavior.
- `{attribute 'qualified_only'}` on enum definitions.
- Online-change-safe initialization logic.
