---
title: "Project: Dykes"
slug: project-dykes
header: laser-hall
summary: "Announcing Dykes: A simple argument parsing library."
author: piper
published: 2025-06-16
---

I'm one of those developers that likes writing little scripts for my own use.
Things like a custom budgeting tool.
Or a word counter.
Actually I write a lot of word counters.
Custom ones for different kinds of writing.

While writing a new word counter, I tried using `Protocol` and `cast` to get a typed namespace from `argparse`.
It was a surprisingly simple solution to a problem.
I idly [mentioned it on fedi](https://ngmx.com/@pathunstrom/114611226324385652) and that started a conversation about arg parsing for simple scripts.

I'm an actual fan of `argparse`.
I think it's great, even with the significant number of warts.
I will also be the first to admit that its learning curve looks like a cliff.

I'm not a fan of tools like `click`, but honestly it's probably a philosophical difference.
It's a good tool that helps build good tools.

Other argparse tools that give type information tend to use metaprogramming and other magic.
I like certain kinds of python magic, but "this does a thing without looking like it's doing a thing" is not my preference.

So I slapped together an idea.

A 13 line module did the same basic thing as my Protocol + cast method,
but gave you a real type out.
It was just an adapter function wrapped around argparse.

One of the regular problems I see in `argparse` wrappers is failure to manage the basic use cases.
Like app descriptions.
Or parameter help strings.

Thus started dykes.

### Project: Dykes

`dykes` is a simple argument parser that does very little "magic".

Build your arguments as a dataclass or `NamedTuple` and pass to the parse_args function.

Get back an instance of your type to use as you see fit.

A sample application:

    from dataclasses import dataclass
    from pathlib import Path
    from typing import Annotated
    
    import dykes
    
    
    @dataclass
    class ExampleApplication:
        """
        This is a sample script that operates on a file on disk.
        """
        path: Annotated[Path, "The paths to operate on."]
        dry_run: bool
        prompt: dykes.StoreFalse
        verbosity: dykes.Count
    
    
    if __name__ == "__main__":
        arguments = dykes.parse_args(ExampleApplication)
        print(arguments)

Skip saying everything twice, get the benefits of type hinting, and enjoy.

You can get it with pip: `pip install dykes`.
Or you can go [grab the code on github](https://github.com/pathunstrom/dykes).

It's MIT licensed, and I hope it helps.

The rest of this article is talking about the moving parts and how I developed them.

### How it got made

Dykes first code contribution started as that thirteen line module.

    import argparse
    from dataclasses import dataclass
    from sys import argv
    from typing import get_type_hints
    
    def simple_args[ArgsType](parameter_definition: type[ArgsType], args: list = None) -> ArgsType:
        if args is None:
            args = argv
        parser = argparse.ArgumentParser()
        hints = get_type_hints(parameter_definition)
        for name, cls in hints.items():
            parser.add_argument(name, type=cls)
        parsed = parser.parse_args(args)
        return parameter_definition(**parsed.__dict__)

Called `simple_parser` at first, I wanted to focus on the big chunk of things I expect to want to put into argparse.

I wanted to be able to use the `type` parameter to coerce strings to relevant types.
I had to support app descriptions and help strings.
Boolean flags and counted flags both needed support out of the box.

The first feature was done in my micro sample.

### Descriptions and Help Strings

The first challenge was app description.

I chose doc strings as my methodology for applying the app description.

Effectively:

    import dataclasses
    from pathlib import Path

    @dataclasses.dataclass
    class Application:
        """
        Counts words in a markdown file, skipping horizontal rules and headings.
        """
        path: Path

Should put the doc string into the `--help` text for the application.

This was relatively simple to produce, as the `inspect` module has a `getdoc` function.
`argparse` automatically strips extra white space and formats it.

Testing this was challenging, though, as you couldn't control how argparse wraps the text.
I have a test case with multiple newlines.

The trick, for those interested, was to check `ArgumentParser.description` instead of the `format_help` output.

Help strings, on the other hand, were a bit more challenging.

I settled on using `typing.Annotated` to produce the right effect.

To update the previous example:

    import dataclasses
    from pathlib import Path
    from typing import Annotated

    @dataclasses.dataclass
    class Application:
        """
        Counts words in a markdown file, skipping horizontal rules and headings.
        """
        path: Annotated[Path, "The file to count."]

To get the annotations, you need two tools: `typing.get_type_hints` and the `Annotated.__metadata__` attribute.

I was already getting the type hints, so I needed to check if a type was `Annotated` and pull off a bare string value.
Usefully, `Annotated` types can be used as the wrapped type directly (and thus needed no special preparation when passed to `argparse`).

Now we got automated help messages that could include human generated text!

### Booleans and Counts

The next step was making sure we could use `bool` to get basic flags works.
I made a decision that a type hint of bool with no default would be a `"store_true"` action.
I also wanted to expose `"store_false"`.

The first thing I did was add a check on the type hint.
If we had a boolean, default the action to `Action.STORE_TRUE`.

That was relatively straight forward, but how about `Action.STORE_FALSE`.
I had written the `Action` `StrEnum` previously that included all the actions for `argparse`.
I already was checking the `__metadata__` for `Annotated` so attempted just plugging an `Action` value in.
With a bit of conditionals to check the `__metadata__` type, I could verify that I had a string enum first and then a string to differentiate between them.
And thus, `Annotated[bool, Action.STORE_FALSE]` was a valid way to describe that.

`"count"` followed a similar path. `Annotated[int, Action.COUNT]` did the right thing.
The problem is the ergonomics weren't great.

I realized I could prebuild these types, since nested `Annotated` types flatten.

So I wrote `dykes.StoreTrue`, `dykes.StoreFalse`, and `dykes.Count`, all of which were `Annotated` with the right type and action.
Annotate them further with a string and you got all the benefits.

All of those features were part of the first "real" release.
Good for the simplest scripts, but missing a key feature: number of arguments.

### NArgs

This took longer than basically every other feature.
Part of it was we were hitting the point where the naive approaches were starting to be a burden.
It also meant making a decision on how to apply extra arguments generally.

First, I handled the most simple case: A positional argument typed `list[T]`.
This meant looking for that list type and changing `nargs` to `+`.
Philosophically, the "simple" case is always "I expect some input, and will fail if I don't get it."

To support the other nargs cases, I decided to lean into the Annotation system.
`ArgumentParser.add_argument` is an extremely flexible function with a bunch of odd foot-guns.
Everything expects to be a keyword, but making sure everything is the "right shape" can cause issues.

I decided on an architecture where I extract the basics from the type system, then use specific Annotated types to produce the arguments.
After I've collected all the arguments, I remove any problematic features (a default for `"store_true"` for example).
Then I pass the dictionary of arguments to `add_argument`.

This meant `NArgs` is a simple type with a single attribute: `value`.
I type hinted this specifically: `Literal["*", "+", "?"] | int`.
Now you just add NArgs to an Annotated type (anywhere in the metadata!) and you'd get the right thing.

A positional argument (so a name with no `Action`) could be typed in a couple of ways:

    @dataclass
    class Application:
        numbers: Annotated[list[float], dykes.NArgs(*)] = dataclasses.field(default_factory=lambda: [1, 2, 3])

This allows you to do any number of arguments (including zero!) and provided a default.
Yes, you use default factory as dataclass expects, and dykes extracts that for you.

You can also get a single optional argument:

    @dataclass
    class Application:
        path: Annotated[Path, dykes.NArgs(?)] = Path(".")

At this point, most of the other actions are available (though may not work _quite_ right.)

As I work through this, I want to support customizing the short and long flags and the other options to add_argument.

Hope some of this rambling is useful to others.
Feel free to peruse the code if you're interested in more details.

Tag me on fedi @pathunstrom@ngmx.com.
