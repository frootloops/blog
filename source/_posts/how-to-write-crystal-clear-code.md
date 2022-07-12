---
title: How to write Crystal Clear Code
date: 2017-11-24 15:10:31
---

Today, we’re going to explore an exciting programming language called Crystal — a compiled Ruby-syntax-like LLVM backend statically typed fully object-oriented language. The reason why I’m so excited about Crystal is the fact that it brings together the benefits of both compiled and dynamic languages.

![](https://miro.medium.com/max/1400/1*rHm4XkrFFvAJDHinuBo3Sg.png)

## Brief history

Crystal was designed and developed by Ary Borenszweig and Juan Wajnerman in 2014. The purpose was to create a language that includes all the benefits of Ruby, but without its associated downsides. Namely, the language must be as elegant and productive as Ruby, but as fast and safe as a compiled/statically typed language.

## Everything is an object
In Crystal, everything is an object. The definition of an object boils down to these points:
* It has a type;
* It can respond to some methods;

## Data types in Crystal

``` crystal
# This is a comment
nil # is used to represent the absence of a value

true  # A Bool that is true
false # A Bool that is false

10    # Int32
1.0   # Float64

'a' # a Char represents a Unicode code point. It occupies 32 bits.
"hello world" # a String represents an immutable sequence of UTF-8 characters.
:hello # a Symbol is a constant that is identified by a name without you having to give it a numeric value.

[1, 2, 3] # Array(Int32)
{1 => "a", 2 => "b"}     # Hash(Int32, String)
x..y  # an inclusive range, in mathematics: [x, y]
{1, "hello", 'x'} # Tuple(Int32, String, Char)
{name: "Crystal", year: 2011} # NamedTuple(name: String, year: Int32)

->(x : Int32) { x.to_s } # Proc(Int32, String)
```

**Symbol** is a number (Int32) but with a human-readable name — for example, `:hello`. Symbols can be used as identifiers or keys of any kind, such as as keys in a hash map.

**Tuple** is a fixed-size, immutable sequence of values. `tuple = {26, "Arsen", '👨‍💻'}` is an example of Tuple(Int32, String, Char). You can also assign labels to values, which will create a NamedTuple. For instance, `tuple = { age: 26, name: "Arsen", avatar: '👨‍💻' }` looks much better and allows us to reach values by using a symbol as a key `tuple[:age] # gives 26`.

**Proc** represents a pointer to a function and a context. The return type is inferred from the proc’s body. `proc = ->(x : Int32) { x.to_s } # is a proc with one input argument and a return value of String type` To invoke a proc, you invoke `aproc.call(1)` method.

A Proc can also be created from an existing method, like so:

``` crystal
def one
  1
end

proc = ->one
proc.call #=> 1
```

## Type system

Despite being a statically type-checked language, Crystal keeps the type system almost invisible for developers. Most of the time, it actually feels like you’re using a dynamically typed language instead.

Crystal achieves this by allowing a variable to have not only one type, but N-types at the same time.

``` crystal
if 1 + 2 == 3
  a = 1
else
  a = "hello"
end

a # : Int32 | String
```

In the example above, the variable a could be an **Int** or a **String**. In Crystal, this is known as a **Union Type**. The most interesting part is how methods are called on a union type. The rule is simple: all types in an union must respond to the method you’re trying to call.

It means that if you try to call **a + 1**, you will get a compile-time error because Strings do not respond to **+** methods.

If you want to call a type-specific method, a runtime check is required:

``` crystal
if a.is_a?(Number)
  a + 1
end
```

## Methods

To define a method, you need to provide a name and specify a list of parameters. You can provide types for the parameters but this is completely optional.

``` crystal
def sum(a : Int32, b : Int32)
  a + b
end

sum(1, 1) # 2 : Int32
```

The return value is the last computed expression within the method. In this case, the return type inferred based on the returning value.

## Nullability

Types in Crystal are non-nilable. To express nullability, Crystal uses a type called **Nil**, which means that if a method returns a value or nil, it will provide a union of some type and Nil.

``` crystal
value = "Hello world"
index = value.index("a") # (Int32 | Nil)
  
if index 
  # index is not a nil
end
```

## Error handling

Lastly, Crystal deals with errors by raising and rescuing exceptions. To raise an exception you use `raise "OH NO!"` method and begin/rescue/end to catch exceptions.

``` crystal
begin
  raise "OH NO!"
rescue
  puts "Rescued!"
end
```

This concludes my quick overview of the Crystal programming language, but if you want to learn more about it, please visit the [official site](http://crystal-lang.org/).

