# Syntax reference

Elixir syntax was designed to have a straightforward conversion to an abstract syntax tree (AST). This means the Elixir syntax is mostly uniform with a handful of "syntax sugar" constructs to reduce the noise in common Elixir idioms.

This document covers all of Elixir syntax constructs as a reference and then discuss their exact AST representation.

## Data types

### Numbers

Integers (`1234`) and floats (`123.4`) in Elixir are represented as a sequence of digits that may be separated by underscore for readability purposes, such as `1_000_000`. Integers never contain a dot (`.`) in their representation. Floats contain a dot and at least one other digit after the dot. Floats also support the scientific format, such as `123.4e10` or `123.4E10`.

### Atoms

Atoms in Elixir start with a colon (`:`) which must be followed by non-combining Unicode characters and underscore. The atom may continue using a sequence of Unicode characters, including numbers, underscore and `@`. Atoms may end in `!` or `?`. See [Unicode Syntax](unicode-syntax.html) for a formal specification. Unicode characters require OTP 20.

All operators in Elixir are also valid atoms. Valid examples are `:foo`, `:FOO`, `:foo_42`, `:foo@bar` and `:++`. Invalid examples are `:@foo` (`@` is not allowed at start), `:123` (numbers are not allowed at start) and `:(*)` (not a valid operator).

If the colon is followed by a double- or single-quote, the atom can be made of any character, such as `:"++olá++"`.

### `true`, `false`, and `nil`

`true`, `false`, and `nil` are reserved words that are represented by the atoms `:true`, `:false` and `:nil` respectively.

### Strings

Strings in Elixir are written between double-quotes, such as `"foo"`. Any double-quote inside the string must be escaped with `\ `. Strings support Unicode characters and are stored in UTF-8 encoding.

Strings are always represented as themselves in the AST.

### Charlists

Charlists in Elixir are written in single-quotes, such as `'foo'`. Any single-quote inside the string must be escaped with `\ `. Charlists are a list of integers, each integer representing a Unicode character.

Charlists are always represented as themselves in the AST.

### Lists, tuples and binaries

Data structures such as lists, tuples, and binaries are marked respectively by the delimiters `[...]`, `{...}`, and `<<...>>`. Each element is separated by comma. A trailing comma is also allowed, such as in `[1, 2, 3,]`.

### Maps and keyword lists

Maps use the `%{...}` notation and each key-value is given by pairs marked with `=>`, such as `%{"hello" => 1, 2 => "world"}`.

Both keyword lists (list of two-element tuples where the first element is atom) and maps with atom keys support a keyword notation where the colon character `:` is moved to the end of the atom. `%{hello: "world"}` is equivalent to `%{:hello => "world"}` and `[foo: :bar]` is equivalent to `[{:foo, :bar}]`. This notation is a syntax sugar that emits the same AST representation. It will be explained in later sections.

## Expressions

### Variables

Variables in Elixir must start with underscore or a non-combining Unicode character that is not in uppercase or titlecase. The variable may continue using a sequence of Unicode characters, including numbers and underscore. Variables may end in `?` or `!`. See [Unicode Syntax](unicode-syntax.html) for a formal specification. Unicode characters require OTP 20.

[Elixir's naming conventions](naming-conventions.html) recommend variables to be in `snake_case` format.

### Non-qualified calls (local calls)

Non-qualified calls, such as `add(1, 2)`, must start with underscore or a non-combining Unicode character that is not in uppercase or titlecase. The call may continue using a sequence of Unicode characters, including numbers and underscore. Calls may end in `?` or `!`. See [Unicode Syntax](unicode-syntax.html) for a formal specification. Unicode characters require OTP 20.

Parentheses for non-qualified calls are optional, except for zero-arity calls, which would then be ambiguous with variables. If parentheses are used, they must immediately follow the function name *without spaces*. For example, `add (1, 2)` is a syntax error, since `(1, 2)` is treated as an invalid block which is attempted to be given as a single argument to `add`.

[Elixir's naming conventions](naming-conventions.html) recommend calls to be in `snake_case` format.

### Operators

As many programming languages, Elixir also support operators as non-qualified calls with their precedence and associativity rules. Constructs such as `=`, `when`, `&` and `@` are simply treated as operators. See [the Operators page](operators.html) for a full reference.

### Qualified calls (remote calls)

Qualified calls, such as `Math.add(1, 2)`, must start with underscore or a non-combining Unicode character that is not in uppercase or titlecase. The call may continue using a sequence of Unicode characters, including numbers and underscore. Calls may end in `?` or `!`. See [Unicode Syntax](unicode-syntax.html) for a formal specification. Unicode characters require OTP 20.

[Elixir's naming conventions](naming-conventions.html) recommend calls to be in `snake_case` format.

For qualified calls, Elixir also allows the function name to be written between double- or single-quotes, allowing calls such as `Math."++add++"(1, 2)`. Operators can be used as qualified calls without a need for quote, such as `Kernel.+(1, 2)`.

Parentheses for qualified calls are optional. If parentheses are used, they must immediately follow the function name *without spaces*.

### Aliases

Aliases are constructs that expand to atoms at compile-time. The alias `String` expands to the atom `:"Elixir.String"`. Aliases must start with an ASCII uppercase character which may be followed by any ASCII letter, number, or underscore. Non-ASCII characters are not supported in aliases.

[Elixir's naming conventions](naming-conventions.html) recommend aliases to be in `CamelCase` format.

### Blocks

Blocks are multiple Elixir expressions separated by newlines or semi-colons. A new block may be created at any moment by using parentheses.

### Left to right arrow

The left to right arrow (`->`) is used to establish a relationship between left and right. The left side may have zero, one or more arguments, the right side is zero, one or more expressions separted by new line. The `->` is always between one of the following terminators: `do`/`end`, `fn`/`end` or `(`/`)`.

It is seen on `case` and `cond` constructs between `do`/`end`:

```elixir
case 1 do
  2 -> 3
  4 -> 5
end

cond do
  true -> false
end
```

Seen in typespecs between `(`/`)`:

```elixir
(integer(), boolean() -> integer())
```

It is also used between `fn/end` for building anonymous functions:

```elixir
fn
  x, y -> x + y
end
```

### Access syntax

Elixir supports `expr[args]`, such as `foo[:bar]`, which is often refered as the access syntax. See the `Access` module for more information.

## The Elixir AST

Elixir syntax was designed to have a straightforward conversion to an abstract syntax tree (AST). Elixir's AST is a regular Elixir data structure composed of the following elements:

  * atoms - such as `:foo`
  * integers - such as `42`
  * floats - such as `13.1`
  * strings - such as `"hello"`
  * lists - such as `[1, 2, 3]`
  * tuples with two elements - such as `{"hello", :world}`
  * tuples with three elements, representing calls or variables, as explained next

The building block of Elixir's AST is a call, such as:

```elixir
sum(1, 2, 3)
```

which is represented as a tuple with three elements:

```elixir
{:sum, meta, [1, 2, 3]}
```

the first element is an atom (or another tuple), the second element is a list of two-item tuples with metadata (such as line numbers) and the third is a list of arguments.

We can retrieve the AST for any Elixir expression by calling `quote`:

```elixir
quote do
  sum()
end
#=> {:sum, [], []}
```

Variables are also represented using a tuple with three elements and a combination of lists and atoms, for example:

```elixir
quote do
  sum
end
#=> {:sum, [], Elixir}
```

You can see that variables are also represented with a tuple, except the third element is an atom expressing the variable context.

Over the next section, we will explore many of Elixir syntax constructs alongside their AST representation.

### Operators

Operators are treated as non-qualified calls:

```elixir
quote do
  1 + 2
end
#=> {:+, [], [1, 2]}
```

Notice that `.` is also an operator. Remote calls use the dot in the AST with two arguments, where the second argument is always an atom:

```elixir
quote do
  foo.bar(1, 2, 3)
end
#=> {{:., [], [{:foo, [], Elixir}, :bar]}, [], [1, 2, 3]}
```

Calling anonymous functions uses the dot in the AST with a single argument, mirroring the fact the function name is "missing" from right side of the dot:

```elixir
quote do
  foo.(1, 2, 3)
end
#=> {{:., [], [{:foo, [], Elixir}]}, [], [1, 2, 3]}
```

### Aliases

Aliases are represented by an `__aliases__` call with each segment separated by dot as an argument:

```elixir
quote do
  Foo.Bar.Baz
end
#=> {:__aliases__, [], [:Foo, :Bar, :Baz]}

quote do
  __MODULE__.Bar.Baz
end
#=> {:__aliases__, [], [{:__MODULE__, [], Elixir}, :Bar, :Baz]}
```

All arguments, except the first, are guaranteed to be atoms.

### Data structures

Remember lists are literals, so they are represented as themselves in the AST:

```elixir
quote do
  [1, 2, 3]
end
#=> [1, 2, 3]
```

Tuples have their own representation, except for two-element tuples, which are represented as themselves:

```elixir
quote do
  {1, 2}
end
#=> {1, 2}

quote do
  {1, 2, 3}
end
#=> {:{}, [], [1, 2, 3]}
```

Binaries have a representation similar to tuples, except they are tagged with `:<<>>` instead of `:{}`:

```elixir
quote do
  <<1, 2, 3>>
end
#=> {:<<>>, [], [1, 2, 3]}
```

The same applies to maps where each pairs is treated as a list of tuples with two elements:

```elixir
quote do
  %{1 => 2, 3 => 4}
end
#=> {:%{}, [], [{1, 2}, {3, 4}]}
```

### Blocks

Blocks are represented as a `__block__` call with each line as a separate argument:

```elixir
quote do
  1
  2
  3
end
#=> {:__block__, [], [1, 2, 3]}

quote do 1; 2; 3; end
#=> {:__block__, [], [1, 2, 3]}
```

### Left to right arrow

The left to right arrow (`->`) is represented similar to operators except that they are always part of a list, its left side represents a list of arguments and the right side is an expression.

For example, in `case` and `cond`:

```elixir
quote do
  case 1 do
    2 -> 3
    4 -> 5
  end
end
#=> {:case, [], [1, [do: [{:->, [], [[2], 3]}, {:->, [], [[4], 5]}]]]}

quote do
  cond do
    true -> false
  end
end
#=> {:cond, [], [[do: [{:->, [], [[true], false]}]]]}
```

Between `(`/`)`:

```elixir
quote do
  (1, 2 -> 3
   4, 5 -> 6)
end
#=> [{:->, [], [[1, 2], 3]}, {:->, [], [[4, 5], 6]}]
```

Between `fn/end`:

```elixir
quote do
  fn
    1, 2 -> 3
    4, 5 -> 6
  end
end
#=> {:fn, [], [{:->, [], [[1, 2], 3]}, {:->, [], [[4, 5], 6]}]}
```

### Access syntax

The access syntax is represented as a call to `Access.get/2`:

```elixir
quote do
  opts[arg]
end
#=> {{:., [], [Access, :get]}, [], [{:opts, [], Elixir}, {:arg, [], Elixir}]}
```

## Syntactic sugar

All of the constructs above are part of Elixir's syntax and have their own representation as part of the Elixir AST. This section will discuss the remaining constructs that "desugar" to one of the constructs explored above. In other words, the constructs below can be represented in more than one way in your Elixir code and retain AST equivalence.

### Integers in other bases and Unicode codepoints

Elixir allows integers to contain `_` to separate digits and provides conveniences to represent integers in other bases:

```elixir
1_000_000
#=> 1000000

0xABCD
#=> 43981 (Hexadecimal base)

0o01234567
#=> 342391 (Octal base)

0b10101010
#=> 170 (Binary base)

?é
#=> 233 (Unicode codepoint)
```

Those constructs exist only at the syntax level. All of the examples above are represented as integers in the AST.

### Optional parentheses

Elixir provides optional parentheses for non-qualified and qualified calls.

```elixir
quote do
  sum 1, 2, 3
end
#=> {:sum, [], [1, 2, 3]}
```

The above is treated the same as `sum(1, 2, 3)` by the parser.

The same applies to qualified calls such as `Foo.bar(1, 2, 3)`, which is the same as `Foo.bar 1, 2, 3`. However, remember parentheses are not optional for non-qualified calls with no arguments, such as `sum()`. Removing the parentheses for `sum` causes it to be represented as the variable `sum`, changing its semantics.

### Sigils

Sigils start with `~` and are followed by a letter and one of the following pairs:

  * `(` and `)`
  * `{` and `}`
  * `[` and `]`
  * `<` and `>`
  * `"` and `"`
  * `'` and `'`
  * `|` and `|`
  * `/` and `/`

After closing the pair, zero or more ASCII letters can be given as a modifier. Sigils are expressed as non-qualified calls prefixed with `sigil_` where the first argument is the sigil contents as a string and the second argument is a list of integers as modifiers:

```elixir
quote do
  ~r/foo/
end
#=> {:sigil_r, [], [{:<<>>, [], ["foo"]}, []]}

quote do
  ~m/foo/abc
end
#=> {:sigil_m, [], [{:<<>>, [], ["foo"]}, 'abc']}
```

If the sigil letter is in uppercase, no interpolation is allowed in the sigil, otherwise its contents may be dynamic. Compare the quotes below for more information:

```elixir
quote do
  ~r/f#{"o"}o/
end

quote do
  ~R/f#{"o"}o/
end
```

### Keywords

Keywords in Elixir are a list of tuples of two elements where the first element is an atom. Using the base constructs, they would be represented as:

```elixir
[{:foo, 1}, {:bar, 2}]
```

However Elixir introduces a syntax sugar where the keywords above may be written as follows:

```elixir
[foo: 1, bar: 2]
```

Atoms with foreign characters in their name, such as whitespace, must be wrapped in quotes. This rule applies to keywords as well:

```elixir
[{:"foo bar", 1}, {:"bar baz", 2}] == ["foo bar": 1, "bar baz": 2]
```

Remember that, because lists and two-element tuples are quoted literals, by definition keywords are also literals (in fact, the only reason tuples with two elements are quoted literals is to support keywords as literals).

### Keywords as last arguments

Elixir also supports a syntax where if the last argument of a call is a keyword then the square brackets can be skipped. This means that the following:

```elixir
if(condition, do: this, else: that)
```

is the same as

```elixir
if(condition, [do: this, else: that])
```

which in turn is the same as

```elixir
if(condition, [{:do, this}, {:else, that}])
```

### `do`/`end` blocks

The last syntax convenience are `do`/`end` blocks. `do`/`end` blocks are equivalent to keywords as the last argument of a function call where the block contents are wrapped in parentheses. For example:

```elixir
if true do
  this
else
  that
end
```

is the same as:

```elixir
if(true, do: (this), else: (that))
```

which we have explored in the previous section.

Parentheses are important to support multiple expressions. This:

```elixir
if true do
  this
  that
end
```

is the same as:

```elixir
if(true, do: (
  this
  that
))
```

Inside `do`/`end` blocks you may introduce other keywords, such as `else` used in the `if` above. The supported keywords between `do`/`end` are static and are:

  * `after`
  * `catch`
  * `else`
  * `rescue`

You can see them being used in constructs such as `receive`, `try`, and others.

## Summary

This document provides a reference to Elixir syntax, exploring its constructs and their AST equivalents.

We have also discussed a handful of syntax conveniences provided by Elixir. Those conveniences are what allow us to write

```elixir
defmodule Math do
  def add(a, b) do
    a + b
  end
end
```

instead of

```elixir
defmodule(Math, [
  {:do, def(add(a, b), [{:do, a + b}])}
])
```

The mapping between code and data (the underlying AST) is what allows Elixir to implement `defmodule`, `def`, `if`, and others in Elixir itself. Elixir makes the constructs available for building the language accessible to developers who want to extend the language to new domains.
