# NativeScript 3.3.0: CSS Parser

CSS parsing of the NativeScript theme, or big chunks of CSS in general stored in app/app.css have been consuming big chunk of apps startup time.

## Baseline Tests
Most grammars, including the CSS3, will strive to fit within O(n) so that optimal parsing speed can be achieved. A state-of-the-art parser will work a few times slower than linearly looping through the characters in the input. The CSS3 parsing is two layered consisting of a tokenizer, that groups character sequences into object (strings, numbers, braces, commas etc.) and a parser that creates an AST like structure.

The times for the iteration over the core.light.css on my mac mini are:
 - Baseline foreach .charCodeAt: 1.76ms.
 - Baseline foreach .charAt: 1.05ms.
 - Baseline foreach indexer: 0.22ms.

After a character is obtained, times to perform basic tests on it are:
 - compareCharIf: 1.09ms.
 - compareCharRegEx: 2.39ms.

So overall one could expect the tokenizer traversal to happen in about 5ms. Further the building of the AST is linear based on the the input tokens stream.

## Existing JavaScript CSS Parsers
On the other hand here are the times that some JavaScript CSS parsing libraries output:
 - [reworkcss/rework](https://github.com/reworkcss/rework): 38.09ms.
 - [PolymerLabs/shady-css-parser](https://github.com/PolymerLabs/shady-css-parser): 16.95ms. (BSD3)
 - [reworkcss/css-parse](https://github.com/reworkcss/css-parse): 30.27ms.
 - [css/gonzales](https://github.com/css/gonzales): 88.90ms.
 - [parserlib](https://github.com/CSSLint/parser-lib): 123.23ms.
 - [css-tree](https://github.com/csstree/csstree): 29.89ms.

Polymer's shady is two times faster than the rework, currently used in {N}, but it is BSD3 and is not compatible with the Apache-2.0 we use to license {N}. The rest does not differ enough, compared to rework, to be considered for proper replacement.

## Writing a Custom Parser
Implementing a CSS3 parser is not technically challenging given [the CSS3 a spec](https://www.w3.org/TR/css-syntax-3/), but may be time consuming. There is a branch within {N}, where some basics of the specs have been implemented in a handwritten parser, and the times yielded by that parser are:
 - nativescript handwritten parser: 7.56ms

The parser is not 100% implemented, it does not escape string characters. For example it will parse properly, but will not replace the `\"` with `"`, at the middle of a `"asd\"asd"` string. So some additional time will be added if it is fully implemented. It will also build an AST that has the raw tokens when building the AST instead of concatenating them back to strings. Rework outputs something like:
``` JSON
{
    "type": "rule",
    "declarations": [
        { "property": "color", "value": "red" },
        { "property": "width", "value": "100px" },
    ]
}
```
While the nativescript handwritten implementation:
``` JSON
{
    "type": "qualified-rule",
    "value": [
        " ", { "type": 1, "text": "color" }, ":", " ", { "type": 1, "text": "red" }, ";",
        " ", { "type": 1, "text": "color" }, ":", " ", { "type": 2, "value": "100", "unit": "px" }
    ]
}
```
What does this mean? The NativeScript framework properties will currently parse the "100px" provided by rework, to a { value: 100, unit: "px" } object. The CSS3 spec however does recognize units and emit unit input tokens, implementing a parser in {N} will let us consume these tokens in the property system saving additional parsing time.

Now adding a layer, over the handmade parser, to convert the AST to the format provided by rework requires some strings to be concatenated and additional JavaScript object tree to be constructed. Measuring it results into:
 - nativescript hand written parser, mapped to rework: 12.12ms.

## Measurements on Hardware
The time with the mapping gets slower, but the AST now can be fed into the NativeScript framework, and this parser can be used to some extent as drop-in replacement for rework. This allows the times to be measured on real device. The following is startup times measured with [nativescript-sdk-examples-ng](https://github.com/NativeScript/nativescript-sdk-examples-ng):
 - Parse times:
    - rework: 207ms.
    - handwritten {N} parser: 78ms.
 - Startup times:
    - rework: 2368ms.
    - handwritten {N} parser: 2511ms.

That's a 150ms. improvement.

Here is the full report for the startup time:
- [rework startup profiling](./reports/android-sdk-ng-nexus5-rework.html)
- [handwritten parser startup profiling](./reports/android-sdk-ng-nexus5-n.html)

Remember that key/value pairs are now back into strings. The property system will have to parse "100%" to units. Also the CSS3 parse time is directly followed by 60ms. create selectors, which once again parses the selectors from strings instead of input tokens. Integrating the input token stream as input for these parsers may further improve times.

Or it may not?

{% include disqus.html %}