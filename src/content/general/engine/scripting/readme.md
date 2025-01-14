**Halo Script** (HSC), also known as **Blam Scripting Language** (BSL), is a
scripting language interpreted by the [Halo engine][engine]. It is used in two main contexts:

1. To control the mission structure and encounters of campaign missions. These scripts are embedded in the map/scenario.
2. In the interactive [developer console][h1/engine/developer-console] of Halo and Sapien, used when debugging and developing maps. This includes files executed at startup like `init.txt` and `editor_init.txt`. Script expressions entered this way are often called **commands**.

Script sources are denoted by the `.hsc` file extension.

# Syntax
As a language, HSC is [Lisp-like][Lisp] and is comprised of [S-expressions][sexp], though it lacks some constructs like looping. This limitation also ensures that scripts never enter into endless loops that block the game's simulation from advancing.

## Expressions
Here's a basic HSC expression:

```hsc
(print "Hello, world!")
```

_Expressions_ are enclosed by parentheses and contain a _function_ name usually followed by _arguments_, all separated by spaces. Note that in the above example, `"Hello, world!"` The function name is always first, so to add two numbers you would write `(+ 1 2)` and **not** `(1 + 2)`. Not all functions need arguments, for example `(garbage_collect_now)`.

Expressions can also evaluate to a value of a certain type, depending on the function used. As a simple example, `(+ 1 2)` evaluates to `3`. You can then use expressions in place of arguments of other expressions:

```hsc
(/ 12 (+ 1 2))
```

If entered into Halo or Sapien's console, this would output "4.000000" as the result of dividing 12 by 3.

You must always balance opening parentheses with closing parentheses and you must use spaces to separate arguments and the function name. Some expressions which would not be allowed are `(/ 12 (+ 1 2)` and `(+(+1 2)3)`. To make it easier to tell if your parentheses are matched and to make your scenario scripts easier to read, it is recommended to use nested indentation for longer expressions like so:

```hsc
; equivalent to writing  (* (+ 10 2) (- 5 1))
(*
  (+ 10 2)
  (- 5 1)
)
```

## Value types
All function arguments and expression results have a particular _type_. The types vary by game, but some common types are:

| Type                          | Example                     |
| ----------------------------- | --------------------------- |
| boolean                       | `true`, `false`             |
| real (32-bit floating point)  | `12.5`, `-1234.0001`        |
| long (32-bit signed integer)  | `1000000000`, `-1000000000` |
| short (16-bit signed integer) | `12`, `-5`                  |
| string                        | `"hello, world!"`           |

Note that the `void` type you may see documented for some functions means the function does not return a value.

See game-specific pages for further reference, e.g. [H1 Scripting][h1/engine/scripting#value-types].

## Comments
You can include comments in your scenario script files:

```hsc
; this is a comment
(print "Hello, world!")

;*
this is a block comment
with multiple lines
*;
(print "Goodbye, world!")
```

## Global variables
A _global variable_ is a value of a fixed [type](#value-types) that can be set and used by any [script thread](#script-threads) at any time. These variables are given a place in memory so they can be used by scripts over the course of gameplay and are saved in checkpoints. HaloScript does **not** have the notion of [local variables][local].

You can only declare globals within scenario scripts. Global variables cannot be declared in the developer console, though you can use or update any that were already initialized by the loaded scenario.

To declare a global variable:
```hsc
(global <global type> <global name> <value(s)>)
; example:
(global boolean completed_objective_b false)
```

To set a global:
```hsc
; mark the objective as completed
(set completed_objective_b true)
```

To use a global:
```hsc
; this will evaluate to true only if both objectives are completed:
(and completed_objective_a completed_objective_b)
; prints the value of a global, useful for troubleshooting in-game:
(inspect completed_objective_b)
```

Globals declared by scenario scripts are called _internal globals_. Halo also has built-in globals that belong to the engine itself called _external globals_. These are for toggling debug features of the engine and testing maps, like `cheat_deathless_player` and `debug_objects`.

## Declaring scripts
Firstly, a bit of terminology. The word "script" is a bit overloaded. Modders often collectively call all scripting in a level its "scripts". This may have been compiled from multiple source `.hsc` files, each of which could also be called a script.

However, HaloScript has a specific concept called a _script_, which is sequence of expressions that are executed according to the script's type by the engine. These can be declared in scenario scripts but not the developer console.

Scripts have the following structure:
```hsc
(script <script type> <return type (static scripts only)> <script name>
  <code>
)

; can be called as (player0) from any other script
(script static "unit" player0
  (unit (list_get (players) 0))
)
; runs every tick to check if players are in a volume and kill them
(script continuous kill_players_in_zone
  (if (volume_test_object kill_volume (list_get (players) 0))
    ; note the call to the player0 static script here
    (unit_kill (player0))
  )
  (if (volume_test_object kill_volume (list_get (players) 1))
    (unit_kill (unit (list_get (players) 1)))
  )
)
```

Script types vary by engine, but the common ones are:

* `continuous`: Runs every tick (simulation frame of the engine).
* `dormant`: Initially asleep until started with `(wake <script_name>)`, runs until there are no instructions left, then stops. Waking a second time will not restart the script.
* `startup`: Begins running at the start of the level and only runs once.
* `static`: Can be called by another script and return a value. Useful for re-usable code -- similar to _methods/functions_ from other programming languages.

See game-specific pages for further reference, e.g. [H1 Scripting][h1/engine/scripting#script-types].

# Mechanics
## Value type casting
HaloScript supports converting data from one type to another, this is called [type casting][cast] or just _casting_. Type casting in HaloScript is done automatically when needed but it's good to keep it in mind as not all types can be converted. Passthrough can be converted to any type and void can be converted to from any type. They are however not the inverse of each other as void destroys the data during conversion.

The rules for object name types are equivalent to the matching object types. Object names can be converted to the equivalent object.

| Target Type     | Source type(s)            |
| --------------- | ----------------          |
| boolean         | real, long, short, string |
| real            | any enum, short, long     |
| long            | short, real               |
| short           | long, real                |
| object_list     | any object or object_name |
| void            | any type                  |
| any type        | passthrough               |
| object          | an other object type      |
| unit            | vehicle                   |

## Script threads
HSC has the notion of threads, i.e. multiple scripts running at the same time (as opposed to waiting until the previous script is done to run). This is not *actual* multithreading (the game still run scripts on a single CPU core). Continuous, dormant, and startup scripts all run in their own threads. Static scripts do not run in their own thread, by virtue of being called by other scripts. For example, a `sleep` within a static script will put the thread of the calling script to sleep. Multiple threads can call the same static script at the same time with no issue. Every thread has its own [stack][stack].

## Implicit returns
Several control structures implicitly return the value of their final expression to the caller. This applies to `if`, `else`, `begin`, `begin_random`, and `cond` blocks, as well as `static` scripts.

```hsc
; The second condition is true, so 6 will be returned from
; the cond block, since it is the final expression.
(cond
  (
    (= 0 1)
    (print "I will never run")
    5
  )
  (
    (= 1 1)
    (print "I will always run!")
    6
  )
  (
    (= 2 2)
    (print "I would run if the code above me hadn't.")
    7
  )
)
```

[local]: https://en.wikipedia.org/wiki/Local_variable
[cast]: https://en.wikipedia.org/wiki/Type_conversion
[stack]: http://en.wikipedia.org/wiki/Call_stack
[sexp]: https://en.wikipedia.org/wiki/S-expression
[Lisp]: https://en.wikipedia.org/wiki/Lisp_(programming_language)
