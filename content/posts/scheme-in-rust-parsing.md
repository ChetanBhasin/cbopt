---
title: "Write Yourself A Scheme In Rust — Parsing (Part 1)"
date: 2019-05-24T23:42:41+02:00
draft: false
toc: false
images:
tags:
  - rust
---

Haskell is one of my favourite programming languages. Whenever someone asks me a way to kick-start their Haskell learning experience, I recommend the WikiBook "Write Yourself A Scheme In 48 Hours."
Recently, I have almost entirely switched to Rust for my day-to-day programming tasks. I thought it might be a good idea to reproduce the Scheme tutorial for Rust. So here we go!

Scheme is one of the most popular dialects of the Lisp programming language — a general-purpose functional programming language. This is going to be a series of blog posts where we will implement a Scheme parser and interpreter. Note that this is not an introductory Rust tutorial. While this will certainly be helpful for new Rust developers, we assume basic Rust understanding. We will, however, point out certain Rust related concepts that might be obvious to more experienced users.

We will start with a naive interpreter, and improve upon it as we go. While I love programming languages and compiler design, I'm sure there will be mistakes — so please feel free to throw suggestion at me. I'll update these articles based on feedback.

## Defining Our Syntax

The first thing that we need to do is be able to read the Lisp syntax for scheme in a format that we can we read and manipulate easily. Fortunately, Scheme does not have a large number of constructs. Being a dialect of Lisp, it also does not differentiate between data and code. If you are not familiar with the idea of code as data, we will address that soon.

First things first, let's define the Lisp syntax as a Rust Enum.

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum LispVal {
    Atom(String),
    Number(i64),
    String(String),
    Boolean(bool),
    List(Vec<LispVal>),
    DottedList(Vec<LispVal>, Box<LispVal>),
}
```

Some of the items in the above Enum are pretty obvious. `Number`, `String`, and `Boolean` types refer to their corresponding types in Rust. For sake of simplicity, we will just use a `u64` as our numeric type for now. `List` also doesn't seem very complicated. It does what you would expect — it holds a list of multiple items from our syntax. `Atom` and `DottedList` are more interesting here, so let's address them.

`Atom` holds any atomic name or symbol that is defined by the user in their program. For example, name of a function will be an atom. It's called atom because that's the smallest level of that symbol and cannot be broken down any further.

A `DottedList` is not that obvious if you're not already familiar with Lisp. The dotted-notation allows you to define your list as pairs using tuples. This is similar to the "cons" notation in languages like Haskell and Scala. We will not address the details of this notation here, but you can learn more about it in [this article](https://www.gnu.org/software/emacs/manual/html_node/elisp/Dotted-Pair-Notation.html).

## Writing A Parser

Now that we have a syntax defined, it's time to write a parser that can read Lisp code and load it up in our Enum representation.
A parser-combinator library will help us parse the code, and combine our smaller parser to build more advanced parsers that span across our entire syntax.

### Picking a library

The Haskell tutorial uses the famous Parsec library. While going through several such libraries for Rust, I found two that I found particularly interesting — Nom and Combine.

While Combine is inspired by Parsec from Haskell, Nom is also not that different from it. The one large difference between the two is that while Nom uses Rust macros for most of the functionality, Combine relies on Rust's trait system for the same.

Personally, I like traits more than macros since macros essentially generate code and I find it easier to reason with code where I can model complex logic in the type-system instead of metaprogramming.
However, Nom had much better documentation available, and it also has a much larger user base. After struggling with Combine for a bit, I gave Nom a try and eventually decided to stick with it. This was mostly because of the documentation and examples though.

### Writing our first parser

Before we get started, let me just put all the imports here. You can add and remove them as you please, but it's always nice to know what the code is using.

```rust
use crate::lisp::*;
use nom::character::complete::{
  alpha1, alphanumeric1, digit1, space0, space1
  };
use nom::*;
use std::iter::FromIterator;
type AppErr<'a> = nom::Err<(&'a str, nom::error::ErrorKind)>;
```

Let's start with the easiest one of all. Let's parse a string.

```rust
named!(
    parse_string<&str, LispVal>,
    do_parse!(
        char!('\"') >>
        value: many0!(none_of!("\"")) >> 
        char!('\"') >>
        (LispVal::String(String::from_iter(value)))
    )
);
```

We used the `named!` macro from Nom here. This will create a function called `parse_string` that will parse our input to a Lisp String. The parser will take `&str` as an input, and produce `LispVal` as the output.

The next thing we do here is to use the `do_parse!` macro. This will let us define our larger parser as a chain of parse operations where inputs can be used from one parser to another. Inside this, we are already using a bunch of other parsers. `char!` parses a single character, `none_of!` parses any character that is not one of the mentioned characters, and `many0!` parses zero or more occurrences of content that the inner parser (`none_of!` here) can parse. 

Since a string will be anything wrapped in quotes, we want to parse the beginning of a quote, a sequence of characters that are not a quote, and an ending code. That is exactly what our parser does. Note how the `>>` here tells Nom to execute one parser after another.

Also note that we store the value of our string in value called, well, `value`. We then use this value to create a Lisp string from our representation.

Now that we have our parser, let's write a quick unit test to see if it works.

```rust
#[test]
fn string_parser_test() {
    let output = parse_string("\"hello\"").unwrap();
    assert_eq!(
        output,
        ("", LispVal::String("hello".to_owned())
    );
}
```

You might have noticed that strings in our example are only reprsented by characters wrapped in quotes. However, what happens when the string itself contains quotes? Usually, we would have something like `\` that could be used for escaping reserved characters. In our parser we haven't done that. Let's get back to it later.

You can work on it as an exercise right away if you like. Here is the code that you can add to your test to validate character escaping.

```rust
let output2 = parse_string(r#""\"hello\""#).unwrap();
assert_eq!(
    output2,
    ("", LispVal::String("\"hello\"".to_owned())
);
```

Now that we have a basic idea of how the Nom API looks like, let's define a few more parsers. Parsing a number sounds like something easy that we can try.

```rust
named!(parse_number<&str, LispVal>, do_parse!(
        number: many1!(digit1) >>
        (LispVal::Number(number.join("").parse::<i64>().unwrap()))
));
```

```rust
#[test]
fn number_parser_test() {
    assert!(parse_number("j5").is_err());
    assert!(parse_number("jlsdf").is_err());
    assert_eq!(parse_number("23").unwrap(), ("", LispVal::Number(23)));
}
```

Our macro calls look similar to the ones that we used for parsing a string. I would like to point out, however, that we can probably define a number parser without using the `do_parse` macro, but it was much easier to store the result of `many1!` in a value that we can read and parse directly in Rust.

You might have also noticed that we introduced two new parsers — `many1` and `digit1` from Nom. `digit1` matches a one or more digit, and `many1` matches one or more occurrences of inner parser.

Let's try to parse our `Atom` values now. Here is another parser written using Nom macros.

```rust
fn match_symbols(input: String) -> LispVal {
    match input.as_str() {
        "#t" => LispVal::Boolean(true),
        "#f" => LispVal::Boolean(false),
        _ => LispVal::Atom(input),
    }
}

named!(parse_atom<&str, LispVal>, do_parse!(
        first: alt!(alpha1 | is_a!(SYMBOL_EXPR)) >>
        rest: many0!(complete!(alt!(alphanumeric1 | is_a!(SYMBOL_EXPR)))) >>
        (match_symbols(format!("{}{}", String::from(first), (String::from_iter(rest)))))
));
```

```rust
#[test]
fn atom_parser_test() {
    assert_eq!(
        parse_atom("$foo").unwrap(),
        ("", LispVal::Atom("$foo".to_owned()))
    );
    assert_eq!(parse_atom("#f").unwrap(), ("", LispVal::Boolean(false)));
}
```

This parser is slightly different in the sense that we match the output of the parser and if the output is `#t` or `#f` we map them to boolean values true and false respectively. If it's none of those, we map them to an atomic value in our Lisp code.

We have also introduced the `alt!` macro now which lets you select one of the multiple specified parsers, and use the one that matches. Rest of the macro code seems fairly intuitive.

Now that we have atoms, lets try something slightly more complicated. Now we can start parsing lists and dotted lists.

```rust
named!(parse_list<&str, LispVal>, do_parse!(
        items: separated_list!(space1, parse_expr) >>
        (LispVal::List(items))
));

named!(parse_quoted<&str, LispVal>, do_parse!(
        char!('\'') >>
        expr: parse_expr >>
        (LispVal::List(vec![LispVal::Atom("quote".to_owned()), expr]))
));

named!(dotted<&str, &str>, do_parse!(space0 >> char!('.') >> space0 >> (".")));

named!(parse_dotted_list<&str, LispVal>, do_parse!(
        exprs: separated_pair!(parse_list, dotted, parse_expr) >>
        ({
            let head = match exprs.0 {
                LispVal::List(v) => v,
                _ => panic!("List parser returned a non-list value")
            };
            LispVal::DottedList(head, Box::new(exprs.1))
        })
));

named!(try_parse_list<&str, LispVal>, do_parse!(
        char!('(') >>
        items: alt!(parse_dotted_list | parse_list) >>
        char!(')') >>
        (items)
));
```

The first parser (`parse_list`) parses a list by using `seperated_list` macro from Nom. The macro allows you to parse any expression (second argument) seperated by first argument (`space1` — that maches one or more space characters).

We are now also parsing things that begin with a quote. In Lisp, these items are also tokens that are represented in our representation.

Note that we also parse dotted lists now. We have the `parse_dotted_list` function defined using macros that uses `dotted` function that is also defined using macros. Our dotted function detects the dotted syntax, and the `parse_dotted_list` function parses the actual list. I felt that this was the most intuitive way of defining a parser for dotted list instead of trying to build the entire parse logic inside of the `do_parse`body.

You might have noticed that we used `parse_expr` while parsing items for our list. Where does this come from? We are now reaching a point where our parsers are becoming recursive. Since a list is basically a sequence of Lisp expressions, and the list itself is an expression, we need a way to write a recursive parser.

We have not yet defined the `parse_expr` function yet, so let's define it.

```rust
named!(parse_expr<&str, LispVal>, alt!(parse_atom | parse_number | parse_string | parse_quoted | try_parse_list));
```

Let's have a few more tests and we are good to go.

```rust
    #[test]
    fn atom_parser_test() {
        assert_eq!(
            parse_atom("$foo").unwrap(),
            ("", LispVal::Atom("$foo".to_owned())
        );
        assert_eq!(parse_atom("#f").unwrap(), ("", LispVal::Boolean(false)));
    }

    #[test]
    fn list_parser_test() {
        assert_eq!(
            parse_list("$foo 42 53").unwrap(),
            (
                "",
                LispVal::List(vec!(
                    LispVal::Atom("$foo".to_owned()),
                    LispVal::Number(42),
                    LispVal::Number(53)
                ))
            )
        );
        assert_eq!(
            parse_list("\"foo\" 42 53").unwrap(),
            (
                "",
                LispVal::List(vec!(
                    LispVal::String("foo".to_owned()),
                    LispVal::Number(42),
                    LispVal::Number(53)
                ))
            )
        );
    }

    #[test]
    fn quoted_parser_test() {
        let output = parse_quoted("'52").unwrap();
        assert_eq!(
            output,
            (
                "",
                LispVal::List(vec![LispVal::Atom("quote".to_owned()), LispVal::Number(52)])
            )
        )
    }
```

Now, this is looking more and more like a useful parser. While this is neither the perfect parser, nor does it satisfy all of supported scheme syntax, it is a very good starting point.

Our parser can now be exported as a public function.

```rust
type AppErr<'a> = nom::Err<(&'a str, nom::error::ErrorKind)>;

pub fn parse_lisp_expr(input: &str) -> Result<(&str, LispVal), AppErr> {
    parse_expr(input)
}
```

We have not addressed several things such as escape characters in strings. Scheme also has vectors that are similar to vectors in Rust — we have not covered them in our parser.
