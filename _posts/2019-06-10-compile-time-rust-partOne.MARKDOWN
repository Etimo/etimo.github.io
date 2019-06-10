---
layout: post
author: Erik Malm
title: Compile time - Rust part 1
image: carRust.jpg
caption: What's different about Rust?
shortexcerpt: What are the basics of Rust? Will this give anyone tetanus?
tag: [featured,compile-time,rust]
categories: [Compiled, Rust,Featured,Compile time]
excerpt_separator: <!--excerpt-->
---
# Rust - Low level language of the future or C--?

Note, this blogpost is in English.  
 This is mainly because writing in English is fun and something I miss doing regularly. Like in the open-source world I am scratching my own itch. Hopefully it is still readable to those not inside my head. 

## Ahead of time

Rust has been a long time coming. Really, a very long time.  
When the Rust programming language started development the author of this blog was unmarried, with no children and six-pack abs. Though I might be lying about the abs.   
Now that I'm older with a beautiful dad-bod Rust has matured too. 

Perhaps the language is even mature enough to build anything from new kernels to webapps in?
Rust has been developed within Mozilla as a replacement language for C and C++, and has much to do with the recent speedups of both Firefox and Dropbox.   
Among it's most tauted features is complete memory safety, checked at compile-time and simple, safe threading. Rust aims squarely at being a low level language with the performance demands that come from this, but unlike C there is an ambition to provide even for the coders out there who are now languishing in Ruby, Python or Javascript.  
  Wouldn't all of them benefit from a safer, faster alternative?
<!--excerpt-->

While my use cases are unlikely to be writing lower lever libraries any time soon, serverless and embedded architectures and compiled web applications could all benefit greatly. 
Where hardware constraints, memory usage and startup time are key features.

Rust could thus benefit the many smaller projects aimed at one of the Raspberry pis that wind up around my household.

Once again Rust will get a non-thorough looking through with any feature that appeals to the writer subjectively being a focus.  
Hopefully this will give a handy future reference to anyone thinking of getting into Rust, or just as a cliff-notes version of the language whenever someone needs a refresher.

## Tooling and eco-system

Rust has an impressive (sometimes opressive) amount of tooling.   
The language itself is delivered through a standardized and near mandatory tool called `rustup`, which makes juggling the myriad release channels of Rust manageable.

Building Rust used to mean compiling newer versions of the compiler using earlier versions,  necessitating a daisy chain of Rust installs until boot-strapping up to the latest version. 
Rustup now offers prebuilt binaries usable in most environments and this is no longer a concern.   

Project and dependency managment is present in the shape of the `Cargo` tool, filling the role of Maven, SBT or Make.
Formatting is slowly being enshrined in code with the RustFmt tool, that is not yet part of the standard Rust installation but planned to get there.   
Since the language enforcing a correct way of formatting code is the best part of Python (anything else you might like is less useful in a team setting!) this is something that will hopefully arrive soon.


Code completion is generally handled in a command line tool called racer, that has hooks into several Emacs, VSCode, Eclipse and Vim plugins.
Those wanting a fuller IDE can get plugins for Eclipse or intellij.  
Even Gnome-builder has Rust support to some extent.

On the Emacs side Spacemacs has an excellent Rust-layer as well with very functional auto-completion, just make sure Racer is properly set up on your system and add the layer to your .spacemacs.

For Vim addicts support for racer is available in Vims YouCompleteMe plugin.
This is as easy as rerunning the YouCompleteMe Python-build script with rustup and cargo installed. In my case:

`python ./YouCompleteMe/install.py --go-completer --clang-completer --racer-completer --js-completer --java-completer`

Along with a Rust-syntax highlighter `Vim.rust` developing is a relatively efficient experience.

An overview of Rust tooling can be found at [Are we IDE yet?](https://areweideyet.com/). 

It's clear that tooling is now high on the Rust projects radar since this overview did not look nearly as rosey about a year ago.

{% include image.html image='crownRust.jpg' caption='Rust is impressively served for tooling, though current development environments do not come without rough edges.' %}
### One Rust, many channels
The eco-system of Rust has a few complexities. Rust is still in development and constantly being released via three different channels - the dependable `stable`channel, the bleeding-edge `nightly` channel and the in-between `Beta`channel. New contents land first in nightly, and is then pushed down to "Beta" every six-weeks.  

While in the `Beta`-channel only major bug fixes will land continously and this will then be pushed into stable every six-weeks. Libraries and tools might have dependencies on ANY of these channels and might also require separate installations for each.
Chances are at some point you will want to develop on Stable while coveting a feature from Nightly and will agonize over the choice or find libraries that depend on one or the other. New releases of Rust are promised not to break backwards compatability and hopefully this is the truth.
Major features of Rust are still in active development, such as the eagerly avaited `async await` pattern.

## Syntax and general concepts

With all that out of the way, what does some basic "Hello world!" code written in rust look like? Well, one version is below:

```rust
fn main() {
  let hello_message = "Hello compiled world!";
  println!("{}",hello_message);
}
```

After working with Pony the syntax of Rust feels deceptively old school. Writing out hello world looks like pretty much like any language from C to Java.

A few things are cleared up though:

- Scoping is explicitly defined using braces (no Pythonic or Pony;ish things here).
- The entry point of the program is a main function.
- The `Main` function does not need to be contained in any sort of container class, so Rust is not a pure object oriented programming language.
- Each line is explicitly terminated with a semi-colon or nothing will compile, unlike for example Scala, Kotlin or Pony.
- The variable is defined as  a "let" implying a low scoped variable. Also there's no type information given when declaring the variable so some type of type inferrence is present.
- The print statement uses a very C-like formatting structure.

There's a bit more to unwind the `!` mark following `println` points out that this is a `Macro` one of Rusts true differentiators, which will be covered a bit below.

Methods and variables are idiomatically defined using snake_case instead of the (obviously!) correct camelCase option, however, when in Rome.
Rust syntax does have some camelCase in it though, types and modules are named using CamelCase so all is not lost.

For the most part Rust hello world looks a lot like C hello world.

## Managed Memory - Ownership,lifetimes and copies

Unlike C, Rust has a very specific ideas about memory and variables in general. Most of these ideas are due to the fact that:

1. Rust is a memory safe language.
1. Rust is not a garbage collected language.

To facilitate this memory safety uses the concept of 'Ownership'.

Ownership entails that:
- Each variable is owned by the scope it is defined in.
- All variables owned by this scope are cleared from memory when the scope ends.
- There can be only one valid pointer to data in memory, owned by a single scope at any one time.
- Whenever a variable is assigned or sent as an argument, the old reference is invalidated and cannot be used or it will throw a compile time error.

This limitation ensures that there is only ever one pointer to data in memory for each variable.  
Since data is never referenced from outside of the scope, it is safe to remove it when the scope reaches conclusion.

Without this limitation the runtime would need to be aware of all references to the same data used in the program, and this would need a higher level process such as a garbage collector to track the life of the data the variable refers to.
This also keeps Rust code thread-safe, enforcing no two threads can refer to the same data at once,

If you're a Swedish speaker and had a look at the Pony compile time (Over here -[Pony part 1]({% post_url 2018-05-08-compile-time-pony %})) the ownership concept very much mimics Ponys use of the capability `iso` where references have to be destroyed on reassignment or passing as an argument.

The below code examples looks at ownership:
```rust
fn main(){
  do_stuff();
}

fn do_stuff() {
  let not_literal = String::from("I am literally not literal"); //Owned by ownstuff here
  println!("This is literally fine: {}",not_literal); //prints fine

  let literal_too = not_literal; //Reassigning this value invalidates the literal variable. 

  //println!("This is literally broken: {}",not_literal); //This line would literally give a compile time error if you uncommented
  //Error: ^^^^^^^^^^^ value used here after move

  call_me_sometime(literal_too); //Transfers ownership to new functions-scope.

  //println!("This is literally the same thing: {}",literal_too); //This line would literally give a compile time error if you uncommented
  //Causes same error:   ^^^^^^^^^^^ value used here after move

  let unique = String::from("I will be copied");
  call_me_sometime(unique.clone());
  println!("Since the data sent was a clone, this is fine: {}",unique);
}

fn call_me_sometime(literal_argument:String) {
  println!("I am literally owned by a new scope: {}",literal_argument);
}

```

This chunk of rust:
1. Creates a `String` type variable.
1. Reassigns it to another variable reference and then passes that new variable to a method.  

Each of these steps disallows the use of the previous variable reference.   
To use data in two places a simple way is to clone it, which can typically be done by calling a `.clone` method on the variable.
Some types with a predefined small size are copied by default in Rust when sent as a method argument,
     such types include all of the Integer and Float types, booleans and characters - when these are passed as an argument they are always copied.

Also notice that when calling a static method defined on the type, this is preceded by `::` in Rust, rather than object methods which are prefixed with `.` . 

### Copies

```rust
/*We are always copied when you send us somewhere*/
u8,u16,u32,u64
i8,i16,i32,i64
char,bool
``` 

### References and mutability

To simplify use of a variable without invalidating the original Rust utilises references.   
In Rust the act of sending a reference into another scope is called "borrowing". 
Creating a reference is done by prefixin an argument or variable with "&". This will essentially act as a pointer to the original variable, which then points to the actual data.
In code this allows simpler handling of ownership then for example sending ownership back and forth between methods, or copying complex data. 

References also allow referencing parts of collection variables, including collections of characters in strings in `Slices`.

```rust

let the_apple = "Apple";
let hello_slice = &hello[0,1]; //First byte of the_apple
let hello_slice = &hello[1,2]; //Second byte of the_apple

```

Immutability is the default in Rust all variables referenced so far in the code have been immutable.
To make a variable mutable the declaration needs to specify it with the `mut` keyword. This also applies to references.
The code below shows a simple example of how to create references to both mutable and immutable variables:

```Rust
const GLOBAL_CONSERVATIVE:&str ="Declare a type, otherwise this won't work."; //Constants is another c-like construct. This constant is in the static scope. It's immutable.

/*
 * Let's look at mutable and immutable references
 */

fn main(){
  let mut mutable_string = String::from("Change is the only true constant.");
  let immutable_string = String::from("Unyielding as the rock.");
  referenced(&mut mutable_string,&immutable_string); //Annotate with mut EVERYWHERE mutability is needed.
  println!("Since ownership is not transfered I can print these from this scope:\n{}, {}", immutable_string,mutable_string);  

}

/**
 * References can be sent without transfering ownership and 
 * mutable references can be mutated. There are limits though.
 */
fn referenced(mutable_ref:&mut String,immutable_ref:&String) {
  mutable_ref.push_str("This is a deep insight at 14.");
  println!("I can be printed here too {}",immutable_ref);
}
```

### Lifetimes - How long do you think you will be needing that reference?

For every reference used Rust compiler infers a lifetime. Rust dosen't allow references that might outlive the thing they're referencing.

Most of the time this is very transparent, but not all of the time.
The simplest way to anger the rust compiler is to write a function returning a reference that contains the thing it references:

```Rust
    fn reference_baddie() -> &str{
      let literal_reference:&str= "I am a literal string"; //We have here a string literal
      &literal_reference
    }

```

Compiling this will give an error to the tune of:

```Rust
| fn reference_baddie() -> &str{
  |                          ^ expected lifetime parameter
```

    The original variable is owned by the function scope and will be deallocated once the function ends. The reference we send out will then be a reference to nothing as soon as it's returned, Rust catches this at compile time.
    It's possible to prevent this deallocation of the literal by defining a static lifetime.  
    Static lifetime string literals are stored in memory at compile time, are protected from modification and are never deallocated from memory, so are always safe to reference.

```Rust
    fn reference_baddie() -> &'static str{
      let literal_reference:&'static str= "I am a literal string"; //We have added a static lifetime to the String literal.
      &literal_reference
    }
```

  The `'static` written after `&` reference is a lifetime definition, `'static` is defines a lifetime of the full runtime of the program.
         Note that this is specified for both the declaration of the string and the return type of the method,
         Rust uses this to check that the lifetime of both the references are the same.

    For most simple lifetimes Rust will simply infer lifetimes correctly but sometimes reference lifetimes need to be annotated explicitly.
    Lifetimes are usually given simple names, such as `'a` and highlight relative dependencies.
    For example all references annotated with `'a` should stay in scope as long as each other. 
    It is also possible to define a relationship between lifetimes where one is defined as smaller then another, a few examples of lifetime annotations in Rust are shown below.

```Rust
    //No specified lifetimes, Rust will infer them as below.
    fn infered_slice(slice_twice:&str) -> &str{
      &slice_twice[0..10]
    }
  //What Rust normally infers, a single lifetime for incoming and outgoing references. 
  //Both references have to have the same scoping to call this. 
  fn specific_slicer<'a>(slice_twice:&'a str) -> &'a str{
    &slice_twice[0..10]
  }

  //Here each parameter is annotated with it's own lifetime ('a and 'b).

  //Won't compile since we specify that the returned
  //reference has lifetime 'b which could be a 
  //longer lifetime then a.

  fn too_specific_slicer<'a,'b>(slice_twice:&'a str) -> &'b str{
    &slice_twice[0..10]
  }
  //Here we also have two different lifetimes a' and b'.
  //This will compile since we've specified a shorter lifetime
  //for 'b than 'a by marking their relation <'a:'b,'b>. 
  fn relational_slicer<'a:'b,'b>(slice_twice:&'a str) -> &'b str{
    &slice_twice[0..10]
  }
}
```

## Macros - pattern matchers for repeatable code

These are the very basics of Rust, signifying a C like syntax, but with added memory managment concepts.

Macros instead step towards the realm of a precompiler, allowing logic to be written at a more abstract layer allowing introspection at the logic trees the Rust compiler is generating.

Because of this macros are compiled at a different stage than standard Rust and has a more restricted set of possibilities than the main language.
Rust itself utilizes the possibilites of macros with many core functions, such as `println!` or `panic!`, both central to the language and implemented as macros.

Macros also seem to be the only way in Rust to create functions with an undefined number of function arguments.

Below is a simple macro that can be dismembered to give a slightly clearer idea of these concepts: 

```Rust
macro_rules! dumbmacro {
  ($( $dumb:expr ),+ ) => {
    let mut counter = 1;
    $(  
        println!("This macro is very unnecessary... {} {}",$dumb,counter.to_string());
        counter=counter+1;

     )*
  }

}
fn main(){
  dumbmacro!("One","Two","Three","Får","Five");
}
```

Quick and very non exhaustive breakdown:
`($( $dumb:expr ),+ )` matches one of more parameters (the ,+ specifies at least one) and bind them to the syntax variable `dumb`. The macro body then:

```Rust
let mut counter = 1; //Standard rust.
$( //Macro syntax for repeated expression.
    println!("This macro is very unnecessary... {} {}",$dumb,counter.to_string()); //The presence of $dumb means rust knows which variable to iterate over.
    counter=counter+1; /

 )*//* means 0 or more members here. No , needed. End of repeated expression.
 ```

As can be seen macro-specific syntax is pretty different from standard Rust but gives an ability to simplify access to code.
Running the macro above with a set of arguments will print  "This macro is very unneccessarry..." for each argument given with a bit extra, as below.


```Bash

This macro is very unnecessary... One 1
This macro is very unnecessary... Two 2
This macro is very unnecessary... Three 3
This macro is very unnecessary... Får 4
This macro is very unnecessary... Five 5
```

Macros extend the toolbox in Rust but are quite separate from most of the code you write but will simplify some use cases massively.

## Enums,Traits, Structs

Rust supports solid generics with repeatable code described as 'Traits'.
More structured collections of data in Rust are defined as structs, and associated methods are defined in an associated `impl` scope.
There is no need to limit method implementations for a struct to a single `impl` block however. The `impl` blocks can even be split among several files and modules.
The fact that 'impl' blocks can be split over several modules sometimes means that imports of a Struct might not import all of it's methods.

For example, a `BufReader` is a buffered reader construct for a stream, this can be imported separately from `BufRead` which implements the `Lines` trait methods for `BufReader`.
The below function reads a file and iterates through the lines in it This code will be referenced again, very soon).

```Rust

use std::io::BufReader;
//use std::io::BufRead;
use std::io::Result;
use std::fs::File;
use std::path::Path;
use std::error::Error;
fn do_the_thing(file_path:String) {
  let path = Path::new(&file_path);
  let reader = match File::open(Path::new(path)) {
    Err(why_man_why) => panic!("I just couldn't do it. I'm sorry. {} {}",file_path,why_man_why.description()),
      Ok(file) => 
        BufReader::new(file)
  };
  let word_up:Vec<Word> = reader.lines() //This is an iterator
    .filter(|line| match line { Ok(_string) => true, _ => false}) //Processing an iterator in this manner in rust is called a :
    .map(|string| {Word::new(string.unwrap())}).collect(); //words

}

```

This code won't compile. The `use std::io::BufReader` does not import the lines method.
Uncommenting the `use std::io:BufRead` line allows the code to compile by bringing the implementation of the lines method into scope.

## Working in Rust with no seatbelts

Sometimes the limits that Rust places on code will indeed restrict functionality, luckily this is something that is wholly understood from by the Rust development community.

There are solutions to most common problems around memory safety, such as wrapping variables in a cell or a refcell to allow more mutable borrows.  
Likewise, solutions exist for sharing data between threads, such as mutex, spinlocks and atomic-reference types.
If some code absolutely needs raw access to a field there is an ability to access raw pointers as well as null if necessary.
Though some of these solutions are outside the spirit of Rust, it is never placed outside of the actual law of the language.

Rusts philosophy allows people to do what they want in the end and this pragmatic approach fact is refreshing, it's as if Javas sun.misc.Unsafe classes had been embraced and codified with best practicers.
Indeed Rust also uses the `unsafe` keyword to allow some of these actions.
As such Rust seems prepared for edge-cases where safety needs to be abandoned for actual project progress, or where low level interactions with hardware or the OS can't be performed with g

## TLDR and Moving forward - Build something Rusty

Rust offers familiar syntax but has a set of concepts at it's core - ownership, borrowing and lifetimes - all in the name of maintaining memory safety.
These impose a few challenges of structure compared to memory unsafe or garbage collected languages. 
Structs and traits offer ways of structuring data and offer defining shared behaviour.
Macros offer an intriguing part of Rust allowing the language to be extended.

To see if these traits add up to a win, tune in for part two of Rust compile time in about a week(-ish) were they'll all be put together into functioning code.
For some last minute grumbling hang around now.

### Pet peeve corner - Shadowing

Rust allows multiple variables with the same name to be declared in the same scope.
This renders the previous variable of that name to become inacessible. This level of shadowing is pretty unique among the common languages. Having recently read the PonyLang documentation, where shadowing is directly pointed out as a known source of bugs, herefore finding it as a proud language feature in Rust was unexpected.
Note that several languages (i.e. Java and Scala ) allow shadowing in nested scopes where the variable in the outer scope can then no longer be referenced without special annotation.
Shadowing could certainly complicate the average debugging session, by a lot. Shadowing does allow the coder to do some clever things that might otherwise be tricky in rust, such as turning immutable variables into mutable ones.

```Rust
fn main(){
let shadow ="No substance";
let mut shadow = shadow; //Now shadow is mutable! Tremble earthlings!
println!("{}",shadow);
let shadow ="To the shadow"; //The previous shadow-variable is now unusable.
println!("{}",shadow);
let shadow = 5; //Now we have an int!
println!("{}",shadow.to_string());

/* Gives output:
No substance
To the shadow
5
     */
}
```

Subjectively nothing else rust does feels as uppsetting and out of character as shadowing.
Shadowing allows developers to not have to come up with new variable names but can make reading code and debugging harder. 
Of course shadowing is wholly optional, but the urge to just not come up with another variation on `foo` is strong.

