# `json2dir`: directory archives, made human-readable, in pure MSBuild

JSON documents → directory trees. Drop-in replacement for [alurm/json2dir](https://github.com/alurm/json2dir) — minus the footguns.

A single MSBuild project file. There is no C#, no inline tasks, no `Exec` and no JSON library. JSON is parsed with .NET regular expressions (balancing groups). Recursion goes through the `<MSBuild>` task calling its own project.

`json2dir` is a tool for converting JSON objects into directory trees using the conversion scheme specified below.

## Table of contents

- [TL;DR](#tldr)
- [Conversion scheme](#conversion-scheme)
- [How it works](#how-it-works)
- [Caveats](#caveats)
- [Requirements](#requirements)

## TL;DR

Let's start with an example.

Assume we have the file named `example-tree.json` in the current directory with the following contents:

```json
{
  "greeting": "Hello, world!",
  "dir": { "subfile": "Content.\n", "subdir": {} },
  "symlink": ["link", "target path"],
  "script":  ["script", "#!/bin/sh\necho Howdy!"]
}
```

Then we run this command:

```sh
cat example-tree.json | MSBUILDENABLEALLPROPERTYFUNCTIONS=1 dotnet msbuild json2dir.proj
```

Here, four entries will be added to the current directory:

- `greeting`: a regular file containing the text `Hello, world!`.
- `dir`: a directory with two entries in it (`subfile` and `subdir`).
- `symlink`: a symbolic link pointing to `target path`.
- `script`: an executable shell script that prints `Howdy!` when run.

JSON is read from stdin via `Console.In.ReadToEnd()`. Pass `-p:In=file.json` to read a file instead, and `-p:Out=dir` to write somewhere other than the current directory. Both are resolved relative to the directory `dotnet msbuild` is run from.

## Conversion scheme

### Objects

Objects represent directories. Keys of objects represent names of files in directories.

#### Examples

An empty directory: `{}`.

A directory with an empty directory named `foo`: `{"foo": {}}`.

### Strings

String values represent contents of files.

#### Examples

A directory with a file named `hello` with the text `Hello, world`: `{"hello": "Hello, world"}`.

### Arrays

Arrays represent symlinks and files with an executable bit set (executable files, scripts).

The first element of the array must be a string.

If the string is `"link"`, the second array element represents the target of the symlink.

If the string is `"script"`, the second array element represents the contents of the executable file.

#### Examples

A symbolic link pointing to the root directory: `["link", "/"]`.

An executable file printing `Hello` when run: `["script", "#!/bin/sh\necho Hello"]`.

## How it works

- `Json2Dir` reads stdin (or the `In` file) and passes it to `Node`.
- `Node` looks at the first character of a value. For `{` it creates a directory and hands the inner text to `Members`. For `"` it writes a file. For `[` it creates a symlink or a script.
- `Members` uses one regex to split off the first `"key": value` pair and the rest of the object. It calls `Node` for the value and itself for the rest.
- A nested value is matched with balancing groups, so brackets inside strings are not counted:

  ```
  [\[{](?>"(?:[^"\\]|\\.)*"|[^\[\]{}"]+|[\[{](?<d>)|[\]}](?<-d>))*(?(d)(?!))[\]}]
  ```

- JSON string escapes (`\"`, `\\`, `\n`, `\uXXXX`, ...) are decoded with `Regex.Unescape`.

## Caveats

- **`MSBUILDENABLEALLPROPERTYFUNCTIONS=1` is required.** Parsing uses only whitelisted property functions. Reading stdin needs `System.Console`, which is not whitelisted. Stock MSBuild cannot write a file byte for byte, though: `XslTransformation` adds a BOM and CRLF, and `WriteLinesToFile` always appends a newline. Symlinks and the executable bit would otherwise need `Exec`. With the variable set, `File.WriteAllText`, `File.CreateSymbolicLink` and `File.SetUnixFileMode` are called directly.
- **`%XX` in strings**: a `%` followed by two hex digits inside a key or a value is decoded as an MSBuild escape (`%41` becomes `A`, `%3B` becomes `;`). A lone `%` is kept as is.
- **Little validation**: the input is assumed to be valid JSON. Keys are checked after unescaping: an empty key, `.`, `..`, a key containing `/` (or, on Windows, any other path separator or volume) or NUL fails the build with an `<Error>` before anything is created for that member. Entries created for earlier members are kept.
- **No deletion**: existing files are overwritten. An existing symlink makes the build fail.
- **Windows**: symlinks need Developer Mode or admin rights. The executable bit is skipped.
- **Time of check, time of use attacks**: when using this to create files for other users, care must be taken to prevent TOCTOU attacks (e.g. with symlinks). No attempt is made to guard against them.

## Requirements

.NET SDK with MSBuild 17+ (tested with .NET SDK 10.0).
