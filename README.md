#Parcoa
## Objective-C Parser Combinators
**Parcoa** is a collection of parsers and parser combinators for Objective-C inspired by Haskell's [Parsec](http://www.haskell.org/haskellwiki/Parsec) package. It is released under a MIT license.

### Pure Parcoa Parsers
A `ParcoaParser *` is a lite wrapper around a function [block](http://developer.apple.com/library/ios/#documentation/cocoa/Conceptual/Blocks/Articles/00_Introduction.html) that takes a single `NSString *` argument, attempts to parse some value from this string, then returns an `OK` or `Fail` result. On success, the parser block returns the parsed *value*, the unconsumed *residual* input, and a message indicating what input would have allowed the parser to consume more characters. On failure, the parser block returns a string description of what it *expected* to find in the input.

![Parser Diagram](https://raw.github.com/brotchie/Parcoa/master/docs/diagrams/parser.png)

Consider a very simple parser that expects the unicode character `'a'`

    ParcoaParser *simpleA = [Parcoa unichar:'a'];

If we pass it the string `"abcd"`, it parses the first character and returns an OK result:

    ParcoaResult *result = [simpleA parse:@"abcd"];
    
    result.isOK == TRUE
    result.value == @"a"

If there's no leading `'a'` then parsing fails:

    ParcoaResult *result = [simpleA parse:@"bcd"];
    
    result.isFail == TRUE

### What the heck is a parser combinator?
Although simple and self-contained, basic Parcoa parsers are useless in isolation. A [parser combinator](http://en.wikipedia.org/wiki/Parser_combinator) is a function that takes one or more parsers and creates a *new* parser with added functionality. Combined parsers can be further combined, enabling complex parsing behaviour via composition of parsing primatives.

For example, instead of a single `'a'` character, we can match any consecutive sequence of a's

```
ParcoaParser *manyA = [Parcoa many:simpleA];

ParcoaResult *result = [manyA parse:@"aaaabcd"];

result.isOK == TRUE
result.value == @[@"a", @"a", @"a", @"a"]]
```

Perhaps we want to match any consecutive sequence of a's *or* the string `"hello"`
```
ParcoaParser *hello = [Parcoa string:@"hello"];
ParcoaParser *manyAConcat = [manyA concat];
ParcoaParser *thisorthat = [manyAConcat or: hello];

ParcoaResult *result = [thisorthat parse:@"helloworld"];

result.isOK == TRUE
result.value == @"hello"

result = [thisorthat parse:@"aaaaaworld"];

result.isOK == TRUE
result.value = @"aaaaa"

```
### A simple JSON parser
Here's a simple JSON parser written using Parcoa. It supports escaped quotes in strings by only support integral number literals.

```objc
ParcoaParser *colon        = [[Parcoa unichar:':'] skipSurroundingSpaces];
ParcoaParser *comma        = [[Parcoa unichar:','] skipSurroundingSpaces];
ParcoaParser *openBrace    = [[Parcoa unichar:'{'] skipSurroundingSpaces];
ParcoaParser *closeBrace   = [[Parcoa unichar:'}'] skipSurroundingSpaces];
ParcoaParser *openBracket  = [[Parcoa unichar:'['] skipSurroundingSpaces];
ParcoaParser *closeBracket = [[Parcoa unichar:']'] skipSurroundingSpaces];

ParcoaParser *quote         = [Parcoa unichar:'"'];
ParcoaParser *notQuote      = [Parcoa noneOf:@"\""];
ParcoaParser *escapedQuote  = [Parcoa string:@"\\\""];
ParcoaParser *stringContent = [Parcoa concatMany:[escapedQuote or: notQuote]];

ParcoaParserForward *json = [ParcoaParserForward forwardWithName:@"json"
                                                         summary:@"json forward declaration"];

ParcoaParser *string  = [stringContent between:quote and: quote];
ParcoaParser *null    = [Parcoa string:@"null"];
ParcoaParser *boolean = [Parcoa bool];
ParcoaParser *integer = [Parcoa integer];
ParcoaParser *pair    = [[string keepLeft: colon] then: json];
ParcoaParser *object  = [[[pair sepBy:comma] between:openBrace   and: closeBrace] dictionary];
ParcoaParser *list    =  [[json sepBy:comma] between:openBracket and: closeBracket];

[json setImplementation:[Parcoa choice:@[object, list, string, integer, boolean, null]]];
```

If we run the parser on some input

    ParcoaResult *result = [json parse:@"[{\"name\" : \"James\", \"age\" : 28, \"active\" : true}]"];
    NSLog(@"%@", result.value);

we get native Objective-C objects as output

    2013-01-03 11:16:46.666 ParcoaJSONExample[20822:c07] (
            {
            Active = 1;
            Age = 28;
            Name = James;
        }
    )

## Installing
You can clone Parcoa from it's [github repository](https://github.com/brotchie/Parcoa) or install it using [CocoaPods](http://cocoapods.org/).

I recommend using CocoaPods because all your linking, build settings, and header includes are set automatically. If you clone from github you'll have to configure these settings manually.
### CocoaPod
Install and configure [CocoaPods](http://cocoapods.org/).

In the root of your project create a file named `Podfile` containing

    xcodeproj '<# YOUR PROJECT #>.xcodeproj'
    pod 'Parcoa', :git => "https://github.com/brotchie/Parcoa.git"
    
then run

    pod install
    
**Note:** After running `pod install` open `<# YOUR PROJECT #>.xcworkspace` in xcode, not `<# YOUR PROJECT #>.xcodeproj`.
    
### Git
Parcoa builds a static library `libParcoa.a`. The library target is in `Parcoa.xcodeproj`. To link Parcoa with your target:

1. Clone the `Parcoa` github repostiory ```git clone https://github.com/brotchie/Parcoa.git```.
2. Drag `Parcoa.xcodeproj` into your XCode project.
3. Add `libParcoa.a` to `Linked Frameworks and Libraries` on your target's `Summary` tab.
4. Ensure `Parcoa (Parcoa)` is in `Target Dependencies` on your target's `Build Phases` tab.
5. Add the Parcoa directory as a recursive path to `User Header Search Paths` and set `Always Search User Paths` to `YES` on your target's `Build Settings` tab.
6. Add `-ObjC` to `Other Linker Flags` on your target's `Build Settings` tab.

### Git Submodule
From the root of your project clone the `Parcoa` repostiory into a git submodule

    mkdir -p Submodules
    git submodule add https://github.com/brotchie/Parcoa.git Submodules/Parcoa
    git submodule update --init
    
then follow the steps under the **Git** heading above.
