---
title: "Why the new js pipeline operator is a terrible idea"
date: 2022-03-31T17:37:03Z
description: "Functional programming styles have some advantages: predictability and great debugability. Those rely on well-defined patterns that implicitly avoid that you end up shooting yourself in the foot. The newly proposed javascript pipe operator comes without this protection and has a good chance of introducing hard-to-understand code and bad bugs. Here's why."
categories: ["functional", "f#", "javascript", "pipe"]
authors: [ "Matthias Kainer", ]
toc: true
dropCap: true
displayInMenu: false
displayInList: true
draft: true
resources:
- name: featuredImage
  src: "js-pipe-op.png"
  params:
    description: "A code example of the js pipe operator from the article"
---

The world of software development evolves at a ludicrous speed. Things considered state of the art 20 years ago, when I started coding, are now considered legacy and bad practice. It's as if doctors would have gone from leeches to gene splicing in half a dozen years. Luckily, I enjoy learning. And I appreciate every new idea and pattern that I get to know with every new language I encounter. The borrow checker and the high level features at high performance from Rust. The easiness of bringing your thoughts about data into code from Python. And the convenience of composing functions with f#, which always makes me feel as if I'm the maestro of the code, timing the different parts of the orchestra ideally to create the most beautiful song. 

And all the things, the big and the small, I love in those languages, I miss in others. When I switch from Rust to another, I miss results and matches. When I switch from Python to another language, I search for their equivalent of NumPy and panda. When I switch from f#, I search for a way to pipe and bind my data.

It seems a few people were coming from f# to javascript and missed that too. Thus, a new feature will be added to the language: The [pipeline operator](https://github.com/tc39/proposal-pipeline-operator). When I first read about this idea, it got me excited. Would I be able to create my workflows in javascript soon? However, after playing around with it, I'm much less enthusiastic and feel that the way this is supposed to be implemented is a mistake. 

Languages have an inherent design that will shape how the language can evolve without contracting itself. Those traits are defined by how the language manages data and state, how it works with types and the path on which it establishes the absence of things—all of those bring together constraints on how one can implement specific patterns in a language. Or, like here, can't.

This very extension looks like a timely topic that will help create more readable code. And on the first look, this is indeed achieved. Rather than a fragment like this `console.log(format(tokenize(readLines(test_data))));`, which we have to read inside-out to understand what it is we want to console.log, we get a statement that we can read almost like a sentence:

```js
test_data
    |> readLines(^)
    |> tokenize(^)
    |> format(^)
    |> console.log(^);
```

This approach of calling function has not been developed by javascript; many different languages have already used it. However, when you have used and learned it there, some shortcomings will lead to bugs that you wouldn't have in other languages.

Showing this is not easy, so I ask you to bear with me. If you have previous experience with functional programming, the tl;dr is that javascript lacks any functor or monad laws. This was the only time I used those words for those who didn't, so there is no reason to worry if such terms scare you.

First, to explain why we will develop a little application that reads the content of a file. Every line in the file represents a specific entry from different sources, with the keys defining the system. We are only interested in the entries from `ELEMENT`. Those entries have the format `ELEMENT[00]=[00]`. The numbers directly after ELEMENT represent the order of the element. The numbers behind the equal sign are a number > 0, illustrating the ceiling-rounded elapsed percentage since the last entry. So, if `ELEMENT01=10` and `ELEMENT02=6`, the total percentage is 16, and the average is 8. As the percentage is ceiling rounded, the total number might go up to over 100%. As the overall system is highly parallel, but for some reason, everybody writes into this one file, it is locked most of the time, and the order of entries is not guaranteed.

Our first task is to create a friendly process display on the console. It should show the percentage, the message "Started..." until it has reached 25%, the message "Running..." until it's at 75%, and "Almost done..." until it's at 100. Once it has reached 100 or more, we should show "Done" and no percentage. Also, we should make sure it's sorted - based on the last few entries, as a future requirement will be to calculate the time remaining.

If you are already familiar with pipes, binding and higher-order functions, please skip the f# chapter and jump directly to [pipes and javascript](#pipes-and-javascript). If you are aware of the proposal and are only here to check what can go wrong, jump ahead to [the final chapter](#then-why-is-it-wrong).

## Pipes and f#

The pipe operator in f# is `|>`. It is defined as

```fsharp
value |> function = function value
```

Thus, it passes the value from the beginning to its end. Accordingly, if the function has another argument, it will be kept and the new element added at the end:

```fsharp
arg2 |> function arg1 = function arg1 arg2 
```

This is an easy thing to do for f#, as all functions are curried by default. That means that the function:

```fsharp
function arg1 arg2 
```

is really

```fsharp
function1 = fun arg1 -> fun arg2 
```

This is nothing you have to think about; the language does it automatically. In addition, you can also use an extension of the pipe to pass multiple arguments by providing a tuple and adding more `|`:

```fsharp
(arg1, arg2) ||> function = function arg1 arg2
```

Let's see how we can use that. We start with some test data and read it line by line:

```fsharp
let test_data = "
ELEMENT01=3
ELEMENT04=4
discarded:incorrect
ELEMENT03=1
foo=4
element06=12
ELEMENT05=12
bar=3
ELEMENT02=12
"

let readLines(v: string) = v.Split[|'\n'|]

test_data
  |> readLines
  |> printfn "%A"
```

If we run this, we can see it works:

```bash
❯ dotnet fsi pipes.fsx
[|""; "ELEMENT01=3"; "ELEMENT04=4"; "discarded:incorrect"; "ELEMENT03=1";
  "foo=4"; "element06=12"; "ELEMENT07=12"; "bar=3"; "ELEMENT02=12"; ""|]
```

Now, the line `test_data |> readLines |> printfn "%A"` is equivalent to `printfn "%A" (readLines test_data)`. However, the second way to format the code has a certain disadvantage. If you want to read it, you have to read it "inside out" to understand it - the subject of the sentence is our test_data. The more nested functions we have, the harder it would be to find the original intent. With pipes, we can start with the subject and then read it like a sentence: `Take the test_data, read the lines, and finally print it`. 

We are far from done, though. Our next task is to tokenize each line. Our first code has to handle an exceptional condition already - tokenization may fail. We have one example already in the test string, `discarded:incorrect`. We will address this by creating a split function that returns a Maybe. A maybe contains some value or none; we can use this to bind it later and only continue with those that have some. 

```fsharp

let split(split: char)(element: string) =
    if (element.Contains(split))
        then Some (element.Split[|split|])
        else None

let tokenize(line: string) =
    line
    |> split '='


test_data
    |> readLines
    |> Seq.map tokenize
    |> printfn "%A"
```

```bash
❯ dotnet fsi pipes.fsx
seq [None; Some [|"ELEMENT01"; "3"|]; Some [|"ELEMENT04"; "4"|]; None; ...]
```

As you can see, both the empty element at the beginning and the incorrect element at index 3 are None. 

I want to add one thing in code here even though it is technically not needed - tokenize currently returns our tokens as an array. I prefer to work with tuples in f#, because it comes with a few benefits (remember the `||>` from above), so we will convert it. We can easily do this with the line `fun line -> Some(line[0], line[1])`, but we can't just pipe it in this case. If we write>

```fsharp
let tokenize(line: string) =
    line
    |> split '='
    |> fun line -> Some(line[0], line[1])
```

the compiler will complain about 
```txt
The type 'Option<_>' does not define the field, constructor or member 'Item'.
```

The compiler is trying to tell us that line is not an array but an Option. Remember the Some/None Maybe that the split function returns. To get rid of the compiler error, we would have to add `.Value` on the line to access the content of the Maybe:

```fsharp
let tokenize(line: string) =
    line
    |> split '='
    |> fun line -> Some(line.Value[0], line.Value[1])
```

This, however, will fail during runtime as the line might be None and thus not have a Value. To make sure we are doing this right, we must bind the value first. Binding in that context means checking it and only executing it when it is some.

We can add this function before the `fun` so that the result will look like this:

```fsharp
let tokenize(line: string) =
    line
    |> split '='
    |> Option.bind (fun line -> Some(line[0], line[1]))
```

Rerunning the code will lead to the following output:

```sh
❯ dotnet fsi pipes.fsx
seq [None; Some ("ELEMENT01", "3"); Some ("ELEMENT04", "4"); None; ...]
```

As you can see, the elements have changed from `[|" ELEMENT01"; "3" |]` to `("ELEMENT01", "3")`, they are tuples now. 

As we will need the binding more often, we will add an infix operator that does the binding for us in the future. The code becomes

```fsharp
let (>>=) m f = Option.bind f m

let tokenize(line: string) =
    line
    |> split '='
    >>= fun line -> Some(line[0], line[1])
```

As we added the new `>>=` operator, we no longer have to write `Option.bind`. Coincidentally, the implementation of `>>=` is almost the same as for `|>` - the only change is that it calls `Option.bind` before calling the passed function.

We haven't talked about the javascript implementation of the pipe until this point. And we won't for some time. But do note that the pipe in f# is merely a starting point for us on our quest to compose functions. It's not just syntactic sugar to avoid reading inside out; it stands on the shoulder of the giant that implicitly and nicely asks us to think in functions and dive into them just as popping up. And naturally, by the laws and the ecosystem that we have around this, we are pushed into a direction that will prevent us later from making mistakes. 

But let's continue for now. First, we will filter the None's out using a built-in function `Seq.choose`, which returns the list comprised of the results "x" for each element where the function returns `Some(x)`. `Seq.choose` can also be viewed as the implementation of `bind` for sequences, and it follows the same rules, making it very predictable for us.

```fsharp
let each(e) = e

test_data
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> printfn "%A"
```

```bash
❯ dotnet fsi pipes.fsx
seq
  [("ELEMENT01", "3"); ("ELEMENT04", "4"); ("ELEMENT03", "1"); ("foo", "4"); ...]
```

Next, we remove all elements we are not interested in and only keep the `ELEMENT`'s. Unfortunately, our test data shows that `element06` is lowercase, so we don't have a guarantee for the casing, so we have to consider that, too. In f#, this means two more functions: one to uppercase the key of the array and another one to check the key for a match:

```fsharp
let keyContains(by: string) ((key,_): string * string) =
    key.Contains(by.ToUpper())

let keyToUpper((key, value): string * string) =
    (key.ToUpper(), value)

test_data
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")
    |> printfn "%A"

```

As you can see, the `keyContains` function is curried by us, which makes it more explicit how this works - the thing piped in is the last argument, and the by before that. There is another thing that you haven't seen, `>>`. This operator allows us to create higher-order functions on the spot. So `Seq.filter` calls a function that calls `keyToUpper` first, and then `keyContains`. Let's run the code and see if it works.

```bash
❯ dotnet fsi pipes.fsx
seq
  [("ELEMENT01", "3"); ("ELEMENT04", "4"); ("ELEMENT03", "1");
   ("element06", "12"); ...]
```

It worked. `element06` is still lowercase, as we uppercased it only in the filter method. But luckily, our functions are so generic that we can move them around quickly to create new functionality. So if you prefer the result consistent with uppercase, you can change the pipeline to:

```fsharp
test_data
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.map keyToUpper
    |> Seq.filter (keyContains "ELEMENT")
    |> printfn "%A"
```

```sh
❯ dotnet fsi pipes.fsx
seq
  [("ELEMENT01", "3"); ("ELEMENT04", "4"); ("ELEMENT03", "1");
   ("ELEMENT06", "12"); ...]
```

It's all just generic functions. You are the maestro putting them in the position you want them when you feel they fit best.

I'll speed up now, as the remainder is pretty straightforward. We sort, convert the value to an integer, and create the sum:

```fsharp
let (>>>=) (m, n) f = Option.bind (fun v -> Option.bind (f v) m) n

let toInt(value: string) =
    match Int32.TryParse value with
    | true, int -> Some int
    | _ -> None

let add (a: int) (b: int) =
    Some(a + b)

let sum(state: int option) (value: int option) =
    (value, state) >>>= add

test_data
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")
    |> Seq.sort 
    |> Seq.map (values >> toInt)
    |> Seq.fold sum (Some 0)
    |> printfn "%A"
```

In the `toInt` function, we create our Option-Maybes again as we can't be sure those are all valid numbers, and for the sum, we extend our binding, allowing us to pass a tuple. This will enable us to use this with the fold function. Running it gives us:

```bash
❯ dotnet fsi pipes.fsx
Some 44
```

Now all that's missing is formatting the result. We are receiving an Option-Maybe; thus, we can create a function that supports binding, and we can be sure this function will get a value:

```fsharp
let (|LessThan|_|) comp value = if value < comp then Some() else None

let format(a: int) =
    Some(
        match a with
        | LessThan 25 -> sprintf "[%d%%] Started..." a
        | LessThan 75 -> sprintf "[%d%%] Running..." a
        | LessThan 100 -> sprintf "[%d%%] Almost done..." a
        | _ -> "Done"
    )

let printValue(value: string) =
    Some(value |> printfn "%s")

test_data
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")
    |> Seq.sort 
    |> Seq.map (values >> toInt)
    |> Seq.fold sum (Some 0)
    >>= format
    >>= printValue
```

```bash
❯ dotnet fsi pipes.fsx
[44%] Running...
```

Works! The pipeline, the core of our application, has gotten quite long. But given it's all just functions, you can create easily add higher-order functions that combine some of them and reduce the size of the pipeline to something like

```fsharp
test_data
    |> extract
    |> transform
    >>= print
```

How will that look in javascript soon, and what does it imply?

## Pipes and javascript

We cannot use Pipes out of the box just yet, as they are only a proposal. For trying them out, we can use a babel plugin. We have to configure it as such to work correctly:

```js
const plugins = [
  [
    "@babel/plugin-proposal-pipeline-operator", {
      topicToken: "^",
      proposal: "hack"
    }
  ],
]
```

There are two options - the topic token we will talk about when we start with the code, and the other specifies the proposal that people handed in. There were four of them; you can read about the history [here](https://github.com/tc39/proposal-pipeline-operator/#why-the-hack-pipe-operator) - hack won, so this is what we are going to use.

We begin again with our 

```js
let test_data = `
ELEMENT01=3
ELEMENT04=4
discarded:incorrect
ELEMENT03=1
foo=4
element06=12
ELEMENT07=12
bar=3
ELEMENT02=12
`

const readLines = (v) => v.split("\n")

test_data
    |> readLines(^)
    |> console.log(^)

```

As we can see, the syntax is similar. Probably the most substantial difference is the `^` character. From a programming paradigm style, this changes the implementation from a [tacit](https://en.wikipedia.org/wiki/Tacit_programming) to a more explicit one. The character is the token we defined in the babel plugin; in other examples, you can see other characters used. To my information, there is no decision on which character is used.

Running this code (after babel transforms it) will output the following result:

```sh
[
  '',
  'ELEMENT01=3',
  'ELEMENT04=4',
  'discarded:incorrect',
  'ELEMENT03=1',
  'foo=4',
  'element06=12',
  'ELEMENT07=12',
  'bar=3',
  'ELEMENT02=12',
  ''
]
```

Similar to f#, we got off with a good start. So, let's continue by tokenizing it. However, unlike in f#, we don't have Maybe or Option type, so we are going to approach the error handling slightly different by filtering the elements that don't have an `=` first and then split them: 

```js
const tokenize = (v) =>
    v
    |> ^.filter(v => v.indexOf("=") > -1)
    |> ^.map(v => v.split("="))


test_data
    |> readLines(^)
    |> tokenize(^)
    |> console.log(^)
```

This is different from how we could approach it in f#, which had the Option type allowing us to map and decide in one go. However, it's not too much of a change either, so we might be inclined to ignore it. Nevertheless, it is crucial to highlight a fundamental difference between the two approaches: In f#, we have a construct that allows us to specify the existence of a value and then bind the values to evaluate if we want to compute them. We consciously decided with the `Seq.choose` statement that we wanted to bind them right away. As an alternative, we could have also written:

```fsharp
    // before
    v
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")

    // after
    v
    |> readLines
    |> Seq.map tokenize
    |> Seq.filter (fun (element) -> 
        element
        >>= fun ((key, _)) -> Some(key)
        >>= toUpper
        >>= contains "ELEMENT"
        |> hasValue)
```

Having the code in such a way allows us to generalize the toUpper/contains functions away from the sequence format. In this specific case, however, it increases the code without improving our sequence. In particular, it reduces the quality as before the `choose`-call gave our compiler a hint that no entry would be optional, whereas this might be the case in the second example. 

Having a good understanding of the choices of Some versus None will help us later, though, and is necessary to keep in mind.

In our javascript example, the code insofar will return:

```sh
[
  [ 'ELEMENT01', '3' ],
  [ 'ELEMENT04', '4' ],
  [ 'ELEMENT03', '1' ],
  [ 'foo', '4' ],
  [ 'element06', '12' ],
  [ 'ELEMENT07', '12' ],
  [ 'bar', '3' ],
  [ 'ELEMENT02', '12' ]
]
```

Now is also an interesting point in time to look at the generated code from babel. If we open the yielded file, this is the result:


```js
const readLines = v => v.split("\n");

const tokenize = v => v.filter(v => v.indexOf("=") > -1).map(v => v.split("="));

_ref2 = tokenize(readLines(test_data)), console.log(_ref2);
```

It tells us two things. First, the tokenize function is more concise without using pipes; the other is that our pipeline itself did become more readable. Rather than the inside-out function in the function call, the pipeline helps us (like with f#) to follow the trail rather than the inside-out function in the function call.

The tokenize function shows us how we probably would have created this in javascript - by chaining the functions on their instance.

Let's continue with implementing the uppercasing and filtering. 

```js

const keyContains = ((by, [key]) => key.indexOf(by.toUpperCase()) !== -1)
const keyToUpper = (([key, value]) => [key.toUpperCase(), value])

test_data
    |> readLines(^)
    |> tokenize(^)
    |> ^.filter(e => e |> keyToUpper(^) |> keyContains("ELEMENT", ^))
    |> console.log(^)
```

We create an arrow function to combine the two methods and use the variable as an entry point for the next nested pipeline series. Again, the same `^` is reused in the inner scope.

When we run this, the result is

```js
[
  [ 'ELEMENT01', '3' ],
  [ 'ELEMENT04', '4' ],
  [ 'ELEMENT03', '1' ],
  [ 'element06', '12' ],
  [ 'ELEMENT07', '12' ],
  [ 'ELEMENT02', '12' ]
]
```

All that's left is sorting it and getting the sum before we can format the output. 

```js
const sort = ([a], [b]) => a.localeCompare(b)

const sumValues = (curr, [_, next]) => 
    curr + Number.parseInt(next)

test_data
    |> readLines(^)
    |> tokenize(^)
    |> ^.filter(e => e |> keyToUpper(^) |> keyContains("ELEMENT", ^))
    |> ^.sort(sort)
    |> ^.reduce(sumValues, 0)
    |> console.log(^)

```

```bash
44
```

The formatting can then be done in an if-else statement:

```js
const format = (value) => {
    if (value < 25) { return `[${value}%] Started... ` }
    else if (value < 75) { return `[${value}%] Running... ` }
    else if (value < 100) { return `[${value}%] Almost done... ` }
    else { return `Done` }
}

test_data
    |> readLines(^)
    |> tokenize(^)
    |> ^.filter(e => e |> keyToUpper(^) |> keyContains("ELEMENT", ^))
    |> ^.sort(sort)
    |> ^.reduce(sumValues, 0)
    |> format(^)
    |> console.log(^)
```

```sh
[44%] Running... 
```

Solution achieved.

## Then... Why is it wrong?

It is not too different from the f# solution, so why complain? It is very similar, but it behaves very differently in some cases. To show how let's fiddle with the input a bit.

```fsharp
let test_data = "
ELEMENT01=3
ELEMENT04=4
discarded:incorrect
ELEMENT03=1
foo=4
element06=not an integer
ELEMENT07=12
bar=3
ELEMENT02=12
"
```

Notice how our lowercase `element06` has received an invalid integer now. So what will happen if we run this?

```sh
❯ dotnet fsi pipes.fsx

❯ 
```

Nothing. There is simply no output. What may appear weird at first makes a lot of sense when we dig deeper. Let's take a look at our pipeline:

```fsharp
success_vars
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")
    |> Seq.sort 
    |> Seq.map (values >> toInt)
    |> Seq.fold sum (Some 0)
    >>= format
    >>= printValue
```

As you might remember, the `>>=` is a bind operation. Bind can be seen as a wrapper function that calls the function if the argument has a value or doesn't if the value is none.

In code, it looks like this:

```fsharp 
let bind f optional =
    match optional with
    | Some x -> f x
    | _ -> None
```

There are additional implementations in f#, for instance, the `choose` function on sequences we have already used. Thus, this behaviour can be assumed as law in every functional realm. Knowing this law and combining it with other elements like lazy evaluations and constructs like `map` or `either` will enable you to write complex computational flows that can be mixed and sequenced freely and predictable.

For instance, if we are interested in the fact that we ended up with `None` in our composition, we can change the code like so:

```fsharp
let printValue (value: string option) =
    match value with
        | Some v -> v |> printfn "%s"
        | _ -> printfn "No value available"

success_vars
    |> readLines
    |> Seq.map tokenize
    |> Seq.choose each
    |> Seq.filter (keyToUpper >> keyContains "ELEMENT")
    |> Seq.sort 
    |> Seq.map (values >> toInt)
    |> Seq.fold sum (Some 0)
    >>= format
    |> printValue
```

The result is now evaluated as:

```sh
❯ dotnet fsi pipes.fsx
No value available
❯ 
```

More importantly, though, when wrapped in a bind, our function will only be called given a valid, existing value. We can trust our computational flow to handle the data; thus, we can focus on the logic.

And this is where things start to fall apart in javascript.

Remember our previous example in javascript, and change the code as we did in f#.

```js
let test_data = `
ELEMENT01=3
ELEMENT04=4
discarded:incorrect
ELEMENT03=1
foo=4
element06=not an integer
ELEMENT07=12
bar=3
ELEMENT02=12
`
test_data
    |> readLines(^)
    |> tokenize(^)
    |> ^.filter(e => e |> keyToUpper(^) |> keyContains("ELEMENT", ^))
    |> ^.sort(sort)
    |> ^.reduce(sumValues, 0)
    |> format(^)
    |> console.log(^)
```

When we execute this, the result will be:

```sh
Done
```

This output is because our computational flow is broken starting from the `reduce` statement. In the `sumValues` function, `element06` will become `NaN`, and all sums after that will be `NaN`. In our format function, as we didn't account for `NaN`, we will return the result of the `else` statement, and the `console.log` is without an option to decide if the result is valid and thus should be printed.

Let's look at the main differences:

- In f#, once we have encountered our first `None`, the `add` is no longer called, as `bind` prevents that. So in javascript, after the current value is `NaN`, addition continues for the other numbers. This is the great advantage of optional types that we have seen in the second chapter.
- In f#, the format function would be called only for valid results from the sum, as we added the binding. In javascript, this is always done.
- As a result, we can either use the binding again for the final print in f# or branch it - based on the outcome. Unfortunately, the javascript code is entirely unable to decide the result at that point.

To achieve the same result, we will first have to invent optional types and binding for javascript. Without it, we cannot create the laws required to prove that our flow will succeed or fail. And thus, like in our example, it will error in a way that will be very hard to trace, debug and understand in a more complex application - leading to the exact opposite of what functional paradigms usually provide.

Introducing the pipe operator people appreciate in other languages as it made their code more readable, robust and predictable, as mere syntactic sugar for functional chaining will be known as a source for buggy and hard-to-understand code in the future.

Suppose you want to add some foreign paradigms to the language. In that case, you can always create extensions in your codebase, as I have done [in the past](https://matthias-kainer.de/blog/posts/why-some-people-prefer-functional-and-others-object-oriented-beginners-guide/#composition-in-the-fp-world). But I would never release them as a framework as long as they don't fully implement the expected constraints, less so propose them as a new language feature.

If you want to use paradigms foreign to the language you get to use, think about other ways of getting there; when longing for functional, for instance, use closure script or elm. Same as you would choose TypeScript if you lack OOP features in Javascript.
