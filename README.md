# LOLCODE Compiler

A small compiler written in Rust that translates a LOLCODE-inspired markup language (`.lol`) into HTML. The language uses readable directives such as `#HAI`, `#MAEK PARAGRAF`, and `#GIMMEH BOLD` to describe a web page, while still allowing ordinary text to be written directly in the source file.

The compiler performs three stages:

1. **Lexical analysis** separates ordinary text from recognized directives.
2. **Syntax analysis** checks the structure of the document and builds a parse tree.
3. **Semantic analysis/code generation** resolves scoped variables and converts the parse tree into HTML.

After generating the HTML file, the program attempts to open it in Google Chrome.

## Features

- Plain text output
- Optional HTML `<head>` and page title
- HTML comments
- Paragraphs
- Bold and italic text
- Unordered lists
- Explicit line breaks
- Embedded audio and video/iframe sources
- Variables with body and paragraph scope
- Case-insensitive compiler directives
- Lexical, syntax, semantic, file, and command-line error reporting
- No third-party Rust dependencies

## Requirements

- A Rust toolchain that supports the Rust 2024 edition
- Cargo (included with a standard Rust installation)
- Windows, if you want the generated page to open automatically
- Google Chrome installed at the path currently hard-coded by the program:

  ```text
  C:\Program Files\Google\Chrome\Application\chrome.exe
  ```

The compiler and HTML generation are otherwise implemented with Rust's standard library. On a system where Chrome is missing or installed elsewhere, the HTML file is still written before the browser-launch step fails.

## Quick start

From the repository root, compile one of the included examples:

```powershell
cargo run -- src/test3.lol
```

This reads `src/test3.lol`, creates or overwrites `src/test3.html`, and then opens that HTML file in Chrome.

To compile another file:

```powershell
cargo run -- path/to/page.lol
```

The input must:

- Exist and be readable.
- Have a `.lol` extension. The extension check is case-insensitive.
- Contain at least one character.
- Form a valid document beginning with `#HAI` and ending with `#KTHXBYE`.

## Minimal example

Create a file named `hello.lol`:

```lolcode
#HAI
#MAEK HEAD
#GIMMEH TITLE My First LOLCODE Page #MKAY
#OIC
#MAEK PARAGRAF
Hello, world! #GIMMEH NEWLINE
#GIMMEH BOLD This text is bold. #MKAY
#OIC
#KTHXBYE
```

Compile it:

```powershell
cargo run -- hello.lol
```

The generated `hello.html` is equivalent to:

```html
<html>
<head><title> My First LOLCODE Page </title></head>
<p>
Hello, world! <br><b> This text is bold. </b></p>
</html>
```

Whitespace around directives is preserved as part of neighboring text, so the exact formatting of the generated file can vary with the source layout. Browsers normally collapse most HTML whitespace when rendering the page.

## Document structure

Every source file follows this overall order:

```text
#HAI
[zero or more comments]
[optional head containing exactly one title]
[body content]
#KTHXBYE
```

A fuller example is available in [`src/test3.lol`](src/test3.lol). [`src/test2.lol`](src/test2.lol) demonstrates variable shadowing between body and paragraph scopes.

### Required program delimiters

| Syntax | Purpose | Generated HTML |
| --- | --- | --- |
| `#HAI` | Starts the document; it must be the first directive | `<html>` |
| `#KTHXBYE` | Ends the document; no tokens may follow it | `</html>` |

### Head and title

The head is optional. If present, it must appear after any leading comments and before body content. It must contain exactly one title.

```lolcode
#MAEK HEAD
#GIMMEH TITLE Page title #MKAY
#OIC
```

This produces:

```html
<head><title> Page title </title></head>
```

The title value must be plain text; nested formatting directives are not accepted.

### Comments

```lolcode
#OBTW This becomes an HTML comment. #TLDR
```

This produces:

```html
<!-- This becomes an HTML comment. -->
```

Comments can appear before the optional head or at body level. They are emitted into the HTML rather than discarded. Comments are not supported inside paragraphs or list items.

### Paragraphs

```lolcode
#MAEK PARAGRAF
Paragraph content goes here.
#OIC
```

This produces a `<p>...</p>` element. Paragraphs can contain plain text, variable references, bold text, italic text, line breaks, audio, video, and lists. Paragraphs cannot be nested.

### Bold and italic text

```lolcode
#GIMMEH BOLD Bold text #MKAY
#GIMMEH ITALICS Italic text #MKAY
```

These produce `<b>...</b>` and `<i>...</i>`, respectively. Their contents must be plain text; formatting constructs cannot be nested inside one another.

### Line breaks

```lolcode
First line #GIMMEH NEWLINE
Second line
```

`#GIMMEH NEWLINE` produces `<br>`. Newlines typed directly in ordinary source text are preserved in the HTML source, but HTML normally collapses them when rendering. Use this directive when a visible line break is required.

### Unordered lists

```lolcode
#MAEK LIST
#GIMMEH ITEM First item #MKAY
#GIMMEH ITEM Second item #MKAY
#OIC
```

This produces:

```html
<ul>
  <li> First item </li>
  <li> Second item </li>
</ul>
```

A list can be empty or contain any number of items. Each item supports exactly one of the following:

- Plain text
- One variable reference
- One bold construct
- One italic construct

The item itself is closed by `#MKAY`. A nested bold, italic, or variable-reference construct also has its own `#MKAY`, so two terminators are required in that case:

```lolcode
#MAEK LIST
#GIMMEH ITEM #GIMMEH BOLD Important item #MKAY #MKAY
#OIC
```

### Audio

```lolcode
#GIMMEH SOUNDZ https://example.com/audio.mp3 #MKAY
```

This generates an audio element whose source is the supplied text:

```html
<audio controls> <source src=" https://example.com/audio.mp3 "></audio>
```

The source must be plain text and is copied directly into the `src` attribute.

### Video and embedded content

```lolcode
#GIMMEH VIDZ https://www.youtube.com/embed/example #MKAY
```

This generates:

```html
<iframe src=" https://www.youtube.com/embed/example "/>
```

The supplied text is copied directly into the iframe's `src` attribute. Use an embeddable URL rather than a normal watch-page URL when working with services such as YouTube.

## Variables and scope

Variables provide compile-time text substitution. A declaration stores a string but emits no HTML:

```lolcode
#I HAZ username #IT IZ Ada #MKAY
```

Use the variable later with:

```lolcode
Hello, #LEMME SEE username #MKAY!
```

The generated HTML contains the stored value in place of the reference.

Variable rules:

- A variable name is trimmed of surrounding whitespace and cannot contain spaces.
- Names are case-sensitive because ordinary text is not normalized.
- A variable must be declared before it is used.
- Declaring the same name again in the same scope replaces its previous value.
- A body-level declaration remains available through the rest of the document.
- A paragraph may declare one local variable, but that declaration must be the first construct immediately after `#MAEK PARAGRAF`.
- A paragraph-local variable shadows a body variable with the same name.
- Variable lookup inside a paragraph checks paragraph scope first, then body scope.
- Paragraph-local variables are discarded at `#OIC` and cannot be used afterward.
- Variable values are plain text and cannot contain another directive.

Example:

```lolcode
#HAI
#I HAZ name #IT IZ Ada #MKAY
Outside: #LEMME SEE name #MKAY
#MAEK PARAGRAF
#I HAZ name #IT IZ Grace #MKAY
Inside: #LEMME SEE name #MKAY
#OIC
Outside again: #LEMME SEE name #MKAY
#KTHXBYE
```

The three references resolve to `Ada`, `Grace`, and `Ada`.

## Complete directive reference

| Directive | Meaning | Closing directive | Allowed context |
| --- | --- | --- | --- |
| `#HAI` | Begin document | `#KTHXBYE` | Entire file |
| `#OBTW` | Begin HTML comment | `#TLDR` | Before the head or at body level |
| `#MAEK HEAD` | Begin HTML head | `#OIC` | Once, before body content |
| `#GIMMEH TITLE` | Begin page title | `#MKAY` | Inside the head |
| `#MAEK PARAGRAF` | Begin paragraph | `#OIC` | Body level |
| `#GIMMEH BOLD` | Begin bold text | `#MKAY` | Body, paragraph, or list item |
| `#GIMMEH ITALICS` | Begin italic text | `#MKAY` | Body, paragraph, or list item |
| `#MAEK LIST` | Begin unordered list | `#OIC` | Body or paragraph |
| `#GIMMEH ITEM` | Begin list item | `#MKAY` | Directly inside a list |
| `#GIMMEH NEWLINE` | Insert a line break | None | Body or paragraph |
| `#GIMMEH SOUNDZ` | Insert audio source | `#MKAY` | Body or paragraph |
| `#GIMMEH VIDZ` | Insert iframe source | `#MKAY` | Body or paragraph |
| `#I HAZ` | Declare a variable | `#IT IZ value #MKAY` | Body, or first construct in a paragraph |
| `#LEMME SEE` | Insert a variable's value | `#MKAY` | Body, paragraph, or list item |

Compiler directives are converted to uppercase during tokenization, so `#hai` and `#HaI` are accepted as `#HAI`. Ordinary text and variable names retain their original case.

## Input and tokenization details

The language treats every `#` as the beginning of a compiler directive. Keep these rules in mind:

- Put whitespace between a directive and its text, URL, variable name, or neighboring directive.
- Do not attach punctuation directly to the end of a directive. For example, write `#MKAY !` rather than `#MKAY!`; the combined token `#MKAY!` is not recognized.
- Multiword directives such as `#MAEK HEAD` and `#GIMMEH BOLD` are recognized as a unit.
- Literal `#` characters in ordinary text are not supported because they are interpreted as directive starts.
- Text between directives is preserved verbatim, including spaces and source line endings.
- The compiler does not escape HTML special characters in text, variable values, comments, or URLs.

Because content is inserted directly into generated markup, compile only trusted input. Characters such as `<`, `>`, `&`, and quotes can alter the resulting HTML structure or attributes.

## Output behavior

The output filename is derived from the input argument by taking everything before its first `.` and appending `.html`.

Common examples:

| Input argument | Output file |
| --- | --- |
| `hello.lol` | `hello.html` |
| `src/test3.lol` | `src/test3.html` |
| `pages/about.lol` | `pages/about.html` |

The compiler creates the output file in the corresponding relative location and overwrites an existing file with the same derived name.

Avoid additional periods in a path or filename. For example, `pages/my.page.lol` is currently shortened at the first period rather than only having its final extension replaced.

Generated markup is intentionally minimal:

- It starts with `<html>` and ends with `</html>`.
- It has no `<!DOCTYPE html>` declaration.
- It does not generate explicit `<body>` tags.
- It does not add CSS, JavaScript, indentation, or HTML escaping.
- Source whitespace is preserved, while compiler directives themselves are replaced by HTML tags or variable values.

## Errors and exit behavior

Errors are written to standard error and normally terminate the process with exit code `1`.

| Error category | Typical cause |
| --- | --- |
| Usage error | No input filename was supplied |
| File error | The source cannot be read or the output cannot be created |
| User error | The file is empty or does not have a `.lol` extension |
| Lexical error | A `#...` directive is not recognized |
| Syntax error | A required opener, closer, text value, or valid body construct is missing or misplaced |
| Static semantic error | A variable is used when it is not available in the current scope |
| Browser-launch error | Chrome is absent from the hard-coded Windows path |

The HTML file is created before Chrome is launched. Therefore, a `Failed to open Chrome` message does not necessarily mean compilation failed; check for the generated `.html` file.

Some malformed inputs near the end of a file may trigger a Rust panic rather than a formatted compiler error because the parser expects another token. Closing every construct and ending with `#KTHXBYE` avoids this condition.

## Project layout

```text
.
├── Cargo.toml       # Rust package metadata
├── Cargo.lock       # Locked dependency graph
└── src/
    ├── main.rs      # Lexer, parser, semantic analyzer, HTML generator, and CLI
    ├── test2.lol    # Variable-scope example
    ├── test2.html   # Generated output for test2.lol
    ├── test3.lol    # Full markup/media example
    └── test3.html   # Generated output for test3.lol
```

All compiler stages currently live in `src/main.rs`:

- `LolcodeLexicalAnalyzer` recognizes directives and retains text tokens.
- `LolcodeSyntaxAnalyzer` validates token order and constructs an internal parse tree.
- `LolcodeSemanticAnalyzer` resolves variables and emits HTML.
- `LolcodeCompiler` coordinates the stages.
- `main` handles the command-line argument, file I/O, output naming, and Chrome launch.

## Development

Check that the project compiles:

```powershell
cargo check
```

Build a debug executable:

```powershell
cargo build
```

Build an optimized executable:

```powershell
cargo build --release
```

Run the resulting executable directly on Windows:

```powershell
.\target\release\lolcode_compiler.exe src\test3.lol
```

There is currently no automated test suite. The checked-in `.lol` examples are useful for manual end-to-end checks; compiling them recreates their corresponding `.html` files and launches each result in Chrome.

## Current limitations

- Browser launching is Windows- and Chrome-specific.
- The Chrome path is not configurable from the command line.
- The output path is based on the first period in the input path.
- Existing output files are overwritten without confirmation.
- HTML content and attribute values are not escaped or validated.
- Bold and italic constructs accept only plain text and cannot be nested.
- List items accept only one plain-text, variable, bold, or italic construct.
- Paragraph-local declarations are limited to one declaration at the very beginning of a paragraph.
- The generated document is minimal HTML and may not pass strict HTML validation.
- The command accepts only one input file per run; additional arguments are ignored.
- No automated tests are currently defined.
