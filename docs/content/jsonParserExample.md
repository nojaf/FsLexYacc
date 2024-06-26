Building JSON parser using FsLex and FsYacc
===========================================

1 Introduction
----------------
FsLexYacc is a solution which allows you to define and generate a lexer and a parser. It's made of two parts:
- FsLex (the lexer generator) 
- FsYacc (the parser generator).  

Parsing is a two phase process. In the first phase the lexer is analyzing text and creates a stream of tokens (or a token list). In the second phase the parser is going through the tokens in the stream and generates output (a syntax tree).

FsYacc (the parser generator) is decoupled from FsLex (the lexer generator) and can accept a token stream from any lexer. The generated parser will require a function which can translate the tokens generated by a lexer into tokens configured in the parser. To avoid additional pointless work if you use FsYacc with FsLex you should define the parser first. This way FsYacc will generate an union type defining all required tokens. Then you can use this union type to generate tokens inside the lexer. So despite the fact that lexing happens before parsing we will define the parser first. It means that in your F# project the generated parser must be placed before the lexer in the list of files to compile.

2 Syntax tree
-----------------
Create a new F# library project and install the FsLexYacc package:

    PM> dotnet add yourproject.fsproj package FsLexYacc

We will start by describing the syntax tree. It will be the result of parsing a text. Add a new file called ``JsonValue.fs`` and paste the following union type definition into it:

    module JsonParsing
    
    type JsonValue = 
        | Assoc of (string * JsonValue) list
        | Bool of bool
        | Float of float
        | Int of int
        | List of JsonValue list
        | Null
        | String of string


Nothing fancy here. ``Assoc`` is simply an object which contains a list of properties (a pair where ``fst`` is a property name and ``snd`` is a value).
``List`` is for JSON arrays. We also have numbers, bool, string and null. It's enough to describe a valid JSON.

3 Parser definition
----------------------

In your project root directory (not in the solution root) create a file called ``Parser.fsy``. Now edit the project file (``.fsproj``): Find the line where JsonValue.fs is included and add the following xml element below it:

    <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
    </FsYacc>

The whole thing should look like this:

    ...
    <ItemGroup>
        <Compile Include="JsonValue.fs" />
        <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
        </FsYacc>
        ...
    <ItemGroup>
    ...

Reload/Open the project and add the following code to ``Parser.fsy``:

    //This parser has been writen with help of "Real world OCaml" book By Yaron Minsky, Anil Madhavapeddy, Jason Hickey (chapter 16)
    %{
    open JsonParsing
    %}
    
    %start start
    
    %token <int> INT
    %token <float> FLOAT
    %token <string> ID
    %token <string> STRING
    %token TRUE
    %token FALSE
    %token NULL
    %token LEFT_BRACE
    %token RIGHT_BRACE
    %token LEFT_BRACK
    %token RIGHT_BRACK
    %token COLON
    %token COMMA
    %token EOF
    
    %type <JsonParsing.JsonValue option> start
    
    %%
    
    start: prog { $1 }
    
    prog:
      | EOF { None }
      | value { Some $1 }
    
    value:
      | LEFT_BRACE object_fields RIGHT_BRACE { Assoc $2 }
      | LEFT_BRACK array_values RIGHT_BRACK { List $2 }
      | STRING { String $1 }
      | INT { Int $1 }
      | FLOAT { Float $1 }
      | TRUE { Bool true }
      | FALSE { Bool false }
      | NULL { Null }
    
    object_fields: rev_object_fields { List.rev $1 };
    
    rev_object_fields:
      | { [] }
      | STRING COLON value { [($1,$3)] }
      | rev_object_fields COMMA STRING COLON value { ($3, $5) :: $1 }
    
    array_values:
      | { [] }
      | rev_values { List.rev $1 }
    
    rev_values:
      | value { [$1] }
      | rev_values COMMA value { $3 :: $1 }

This file is describing parsing rules and tokens. When you build the project, a new file called ``Parser.fs`` will be created in the project root directory. Include it in your project.

Lets take a closer look at the parser definition (``Parser.fsy)``. At the very top of the file there is a section for ``open`` statements. You can open any namespace or module you want. We need only ``JsonParsing`` where we defined our ``JsonValue`` structure. All open statements should be between `` %{`` and ``%}``.

Next in line 6 we state what is an entry rule for our parser. ``start ``will be the name of the rule which will be exposed by the parsing module. It has to correspond to a name of a rule (in our case line 27).

Lines 8-21 define the list of tokens. INT, FLOAT, STRING carry a value with them.

Then in line 23 we define what will be the result of parsing. In our case it will be a JSON syntax tree defined by means of ``JsonValue``. The type is actually ``JsonValue option``. For an empty text/file the parser will return ``None``.

Everything that follows ``%%`` are parsing rules. Rules are made of lists of productions. You can reference one rule from the other which will allow you to define nested parsers (we will use it for objects and lists).

Our entry point (line 27) simply calls the ``prog ``rule and returns it's result. Everything between curly braces is ordinary F# code.

``prog ``(line 29) has two productions. The first rule says that if the first token is ``EOF ``then return ``None``. The second rule says: execute the ``value`` rule and return ``Some`` containing it's result. Productions are processed one after another from top to bottom.

On line 33 we define the main rule which will parse JSON values. Each rule has a name and a list of productions. Each production starts with a token pattern and contains a result block ``{}`` which states what should be returned if this pattern is matched. A result block contains ordinary F# code.

Rules can reference each other. Like at line 34 where the pattern says: Match a left brace, whatever will be matched by ``object_fields rule`` and a right brace. Now, what is this ``Assoc $2``? ``Assoc`` is a JsonValue union type that we create and ``$2`` corresponds to the value matched at the second position in the pattern which in this case is a list of properties.

If you look at the ``object_fields`` rule, you'll notice it actually calls ``rev_object_fields`` and reverses the order of results.  ``rev_object_fields`` is trying to match the list of properties, but it will collect them in the wrong order. THe first production says that if the token stream is empty (there is nothing between the braces) then return the empty list. Please note, we match an empty stream by not providing any pattern. The second rule is more interesting. It says that if we encounter a string then a colon and anything that matches the ``value`` rule we should return a singleton list containing a pair. The first element in the pair is the matched string (``$1``) and the second is the matched value (``$3``). This production will be used for objects that have one element or for the last property of the list (there is no comma at the end). The third rule contains two references to other rules. First we match any list of values, then a comma, a string, a colon and a ``value``. The result is a list where head is the matched property (tokens on position 3-5) and the tail is made of other matched properties. This production is for all properties which are followed by a comma. The rules for arrays are very similar (even simpler).

That's it. We can now start building the lexer. You should be able to compile the project and see the generated parser in ``Parser.fs``. It looks ugly but fortunately we will always just deal with the grammar description.

4 Lexer
-----------

In your project root directory (not in the solution root) create a file called ``Lexer.fsl``. Now edit the project file (``.fsproj``). Find the line where ``JsonValue.fs`` is included and add this xml elment below it:

    <FsLex Include="Lexer.fsl">
        <OtherFlags>--unicode</OtherFlags>
    </FsLex>

The whole thing should look like this:

    ...
    <ItemGroup>
      <Compile Include="JsonValue.fs" />
      <FsYacc Include="Parser.fsy">
        <OtherFlags>--module Parser</OtherFlags>
      </FsYacc>
      <FsLex Include="Lexer.fsl">
        <OtherFlags>--unicode</OtherFlags>
      </FsLex>
    ...

Reload/Open the project and add the following code to ``Lexer.fsl:``

    //This lexer has been writen with help of "Real world OCaml" book By Yaron Minsky, Anil Madhavapeddy, Jason Hickey (chapter 16)
    {
    
    module Lexer
    
    open FSharp.Text.Lexing
    open System
    open Parser
    
    exception SyntaxError of string
    
    let lexeme = LexBuffer<_>.LexemeString
    
    let newline (lexbuf: LexBuffer<_>) = 
      lexbuf.StartPos <- lexbuf.StartPos.NextLine
    }
    
    let int = ['-' '+']? ['0'-'9']+
    let digit = ['0'-'9']
    let frac = '.' digit*
    let exp = ['e' 'E'] ['-' '+']? digit+
    let float = '-'? digit* frac? exp?
    
    let white = [' ' '\t']+
    let newline = '\r' | '\n' | "\r\n"
    
    rule read =
      parse
      | white    { read lexbuf }
      | newline  { newline lexbuf; read lexbuf }
      | int      { INT (int (lexeme lexbuf)) }
      | float    { FLOAT (float (lexeme lexbuf)) }
      | "true"   { TRUE }
      | "false"  { FALSE }
      | "null"   { NULL }
      | '"'      { read_string "" false lexbuf } 
      | '{'      { LEFT_BRACE }
      | '}'      { RIGHT_BRACE }
      | '['      { LEFT_BRACK }
      | ']'      { RIGHT_BRACK }
      | ':'      { COLON }
      | ','      { COMMA }
      | eof      { EOF }
      | _ { raise (Exception (sprintf "SyntaxError: Unexpected char: '%s' Line: %d Column: %d" (lexeme lexbuf) (lexbuf.StartPos.Line + 1) lexbuf.StartPos.Column)) }
    
    
    and read_string (str: string) (ignorequote: bool) =
      parse
      | '"'           { if ignorequote then (read_string (str + "\\\"") false lexbuf) else STRING (str) }
      | '\\'          { read_string str true lexbuf }
      | [^ '"' '\\']+ { read_string (str + (lexeme lexbuf)) false lexbuf }
      | eof           { raise (Exception ("String is not terminated")) }

When you build the project, a new file called ``Lexer.fs`` will be created in the project root directory. Include it in your project after ``Parser.fs``.

The first part of a lexer is simply F# code enclosed in ``{}``. It defines a module, opens namespaces and defines helper functions. The ``lexeme`` function will extract the matched string from the buffer. The ``newline`` function updates the buffer position to skip new line characters. Notice that we open the <em>Parser</em> module which contains the union type with the tokens. We will use them in our productions.

Next we have a list of named regular expressions which we can use later. We can also use one regular expression within another by simply referring to it by name. See line 22. The ``float`` expression is referencing ``digit``, ``frac`` and ``exp``. Space in an expression means concatenation or "and then" if you will. For example, the ``frac`` expression at line 20 means: dot character then any number of digits. Characters in single quotes are matched literally, for example '-' or '.'. If there are multiple expressions enclosed in ``[]`` at least one of them must be matched. You can also use ranges: ``['0'-'9'] ['a'-'z'] ['A'-'Z']``.  
The meaning of repetition patterns is as follows:

* ? - 0 or 1
* \+ - 1 or more
* \* - 0 or more

After the list of regular expression patterns we have lexing rules. Each rule contains a list of patterns with productions. Same as in the parser one rule can reference another. You can also pass arguments between rules.

Take a look at a productions. The left side is a pattern and the right side is F# code which will be executed when the pattern is matched. The F# code must return a value (the same type of value across all productions within one rule). In our case, we're returning tokens, which will be later used by the parser.

The ``read_string`` rule takes two arguments: ``str`` which is an accumulator, and a flag ``ignorequote`` which will make the rule ignore the next quote. This way we can process strings which contain escaped quotes like: "abc\"df".

You should be able to build project and preview the generated lexer in ``Lexer.fs``.

5 Program
-------------

The last piece of the puzzle is to write a program which will use our parser. Look at code below. The ``parse`` function will take a json string, parse it and return the syntax tree. You can see the result in debug:

    module Program
    open FSharp.Text.Lexing
    open JsonParsing
    
    [<EntryPoint>]
    let main argv =
        let parse json = 
            let lexbuf = LexBuffer<char>.FromString json
            let res = Parser.start Lexer.read lexbuf
            res
        let simpleJson = @"{
                  ""title"": ""Cities"",
                  ""cities"": [
                    { ""name"": ""Chicago"",  ""zips"": [60601,60600] },
                    { ""name"": ""New York"", ""zips"": [10001] } 
                  ]
                }"
        let (Some parseResult) = simpleJson |> parse

        0

You can also add a ``print`` function to JsonValue and show the result of parsing in the console:

    type JsonValue = 
        | Assoc of (string * JsonValue) list
        | Bool of bool
        | Float of float
        | Int of int
        | List of JsonValue list
        | Null
        | String of string
        //below function is not important, it simply prints values 
        static member print x = 
            match x with
            | Bool b -> sprintf "Bool(%b)" b
            | Float f -> sprintf "Float(%f)" f
            | Int d -> sprintf "Int(%d)" d
            | String s -> sprintf "String(%s)" s
            | Null ->  "Null()"
            | Assoc props -> props 
                             |> List.map (fun (name,value) -> sprintf "\"%s\" : %s" name (JsonValue.print(value))) 
                             |> String.concat ","
                             |> sprintf "Assoc(%s)"
            | List values -> values
                             |> List.map (fun value -> JsonValue.print(value)) 
                             |> String.concat ","
                             |> sprintf "List(%s)"
                             |> sprintf "List(%s)"

And then change program to print the parsing result in the console. Add this line after line 18:

    printfn "%s" (JsonValue.print parseResult)
