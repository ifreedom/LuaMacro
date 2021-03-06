## LuaMacro - a macro preprocessor for Lua

This is a library and driver script for preprocessing and evaluating Lua code. Lexical macros can be defined, which may be simple C-preprocessor style macros or macros that change their expansion depending on the context.

It is a new, rewritten version of the [Luaforge](http://luaforge.net/projects/luamacro/) project of the same name, which required on the [token filter patch](http://www.tecgraf.puc-rio.br/~lhf/ftp/lua/#tokenf) by Luiz Henrique de Figueiredo. This patch allowed Lua scripts to filter the raw token stream before the compiler stage. Within the limits imposed by the lexical filter approach this worked pretty well.  However, the token filter patch is unlikely to ever become part of mainline Lua, either in its original or [revised](http://lua-users.org/lists/lua-l/2010-02/msg00325.html) form. So the most portable option becomes precompilation, but Lua bytecode is not designed to be platform-independent and in any case changes faster than the surface syntax of the language. So using LuaMacro with LuaJIT would have required re-applying the patch, and remaining within the ghetto of specialized, experimental use.

This implementation uses a [LPeg](http://www.inf.puc-rio.br/~roberto/lpeg.html) lexical analyser originally by [Peter Odding](http://lua-users.org/wiki/LpegRecipes) to tokenize Lua source, and builds up a preprocessed string explicitly, which then can be loaded in the usual way. This is not as efficient as the original, but it can be used by anyone with a Lua interpreter, whether it is Lua 5.1, 5.2 or LuaJIT 2. An advantage of fully building the output is that it becomes much easier to debug macros when you can actually see the generated results. (Another example of a LPeg-based Lua macro preprocessor is [Luma](http://luaforge.net/projects/luma/))

It is not possible to discuss macros in Lua without mentioning Fabien Fleutot's [Metalua](metalua.luaforge.net/) which is an alternative Lua compiler which supports syntactical macros that can work on the AST (Abstract Syntax Tree) itself of Lua. This is clearly a technically superior way to extend Lua syntax, but again has the disadvantage of being a direct-to-bytecode compiler. (Perhaps it's also a matter of taste, since I find it easier to think about extending Lua on the lexical level.)

My renewed interest in Lua lexical macros came from some discussions on the Lua mailing list about numerically optimal Lua code using LuaJIT. We have been spoiled by modern optimizing C/C++ compilers, where hand-optimization is often discouraged, but LuaJIT is new and requires some assistance. For instance, unrolling short loops can make a dramatic difference, but Lua does not provide the key concept of constant value to assist the compiler. So a very straightforward use of a macro preprocessor is to provide named constants in the old-fashioned C way. Very efficient code can be generated by generalizing the idea of 'varargs' into a statically-compiled 'tuple' type.

    tuple(3) A,B

The assigment `A = B` is expanded as:

    A_1,A_2,A_3 = B_1,B_2,B_3

I will show how the expansion can be made context-sensitive, so that the loop-unrolling macro `do_` changes this behaviour:

    do_(i,1,3,
        A = 0.5*B
    )

expands to:

    A_1 = 0.5*B_1
    A_2 = 0.5*B_2
    A_3 = 0.5*B_3

Another use is crafting DSLs, particularly for end-user scripting. For instance, people may be more comfortable with `forall x in t do` rather than `for _,x in ipairs(t) do`; there is less to explain in the first form and it translates directly to the second form. Another example comes from this common pattern:

    some_action(function()
      ...
    end)

Using the following macro:

    def_ block (function() _END_CLOSE_

we can write:

    some_action block
       ...
    end

A serious criticism of lexical macros is that they don't respect the scoping rules of the language itself. They are always ruthlessly global. Bad experiences with the C preprocessor has practically put the lexical macro in the same box as gotos; to avoid them is putting the ugly history of computing behind you.  For me, a more serious charge against 'macro magic' is that it can lead to a private dialect of the language (the original Bourne shell was written in C 'skinned' to look like Algol 68.)  This often indicates a programmer uncomfortable with a language who wants it to look like something more familiar. Relying on a preprocessor may mean that programmers need to immerse themselves in the idioms of the new language.

That being said, macros can extend a language so that it can be more expressive for a particular task, particularly if the user is not a professional programmer. This release of LuaMacro introduces lexically-scoped macros, which helps with the first criticism.

### Basic Macro Substitution

To install LuaMacro, expand the archive and make a script or batch file that points to `luam.lua`, for instance:

    lua /home/frodo/luamacro/luam.lua %*

(Or '$*' if not on Windows.) Then put this file on your executable path.

Any Lua code loaded with `luam` goes through four distinct steps:

  * loading and defining macros
  * preprocessing
  * compilation
  * execution

The last two steps happen within Lua itself, but always occur, even though the Lua compiler is fast enough that we mostly do not bother to save the generated bytecode.

For example, consider this `hello.lua`:

    print(HELLO)

and `hello-def.lua`:

    local macro = require 'macro'
    macro.define 'HELLO "Hello, World!"'

To run the program:

    $> luam -lhello-def hello.lua
    Hello, World!

So the module `hello-def.lua` is first loaded (compiled and executed, but not preprocessed) and only then `hello.lua` can be preprocessed and then loaded.

Naturaly, there are easier ways to use LuaMacro, but I want to emphasize the sequence of macro loading, preprocessing and script loading.

`hello2.lua` is a more sensible first program:

    require_ 'hello-def'
    print(HELLO)

You cannot use the Lua `require` function at this point, since `require` is only executed when the program starts executing and we want the macro definitions to be available during the current compilation. `require_` is the macro version, which loads the file at compile-time.

There is also `include_`, which is analogous to `#include` in `cpp`. It takes a file path in quotes, and directly inserts the contents of the file into the current compilation. Although tempting to use, it will not work here because again the macro definitions will not be available at compile-time.

`hello3.lua` fits much more into the C preprocessor paradigm, which uses the `def_` macro:

    def_ HELLO "Hello, World!"
    print(HELLO)

(Like `cpp`, such macro definitions end with the line; however, there is no equivalent of `\` to extend the definition over multiple lines.)

`luam` has a `-d` flag, meaning 'dump', which is very useful when debugging the output of the preprocessing step.

`def_` works pretty much like `#define`, for instance, `def_ SQR(x) ((x)*(x))`. A number of C-style favourites can be defined, like `assert_` using `_STR_`, which is a predefined macro that 'stringifies' its argument.

    def_ assert_(condn) assert(condn,_STR_(condn))

`def_` macros are _lexically scoped_:

    local X = 1
    if something then
        def_ X 42
        assert(X == 42)
    end
    assert(X == 1)

LuaMacro keeps track of Lua block structure - in particular it knows when a particular lexical scope has just been closed.  This is how the `_END_CLOSE_` built-in macro works

    def_ block (function() _END_CLOSE_

    my_fun block
      do_something_later()
    end

When the current scope closes with `end`, LuaMacro appends the necessary ')' to make this syntax valid.

A common use of macros in both C and Lua is to inline optimized code for a case. The Lua function `assert()` always evaluates its second argument, which is not always optimal:

    def_ ASSERT(condn,expr) if condn then else error(expr) end

    ASSERT(2 == 1,"damn! ".. 2 .." is not equal to ".. 1)

If the message expression is expensive to execute, then this can give better performance at the price of some extra code. `ASSERT` is now a statement, not a function, however.

### Using macro.define

`macro.define` is less convenient than `def_` but much more powerful. The extended form allows the substitution to be a _function_ which is called in-place at compile time:

    macro.define('DATE',function()
        return '"'..os.date('%c')..'"'
    end)

Any text which is returned will be tokenized and inserted into the output stream. The explicit quoting here is needed to ensure that `DATE` will be replaced by the string "04/30/11 09:57:53".  ('%c' gives you the current locale's version of the date; for a proper version of this macro, best to use `os.date` [with more explicit formats](http://www.lua.org/pil/22.1.html) .)

This function can also return nothing, which allows you to write macro code purely for its _side-effects_.

Non-operator characters like `@`,`$`, etc can be used as macros. For example, say you like shell-like notation `$HOME` for expanding environment variables in your scripts.

    macro.define '$(x) os.getenv(_STR_(x))'

A script can now say `$(PATH)` and get the expected expansion, Make-style. But we can do better and support `$PATH` directly:

    macro.define('$',function(get)
        local var = get:name()
        return 'os.getenv("'..var..'")'
    end)

If a macro has no parameters, then the substitution function receives a 'getter' object. This provides methods for extracting various token types from the input stream. Here the `$` macro must be immediately followed by an identifier.

We can do better, and define `$` so that something like `$(pwd)` has the same meaning as the Linux shell:

    macro.define('$',function(get)
       local t,v = get()
       if t == 'iden' then
          return 'os.getenv("'..v..'")'
       elseif t == '(' then
          local rest = get:upto ')'
          return 'os.execute("'..tostring(rest)..'")'
       end
    end)

(The getter `get` is callable, and returns the type and value of the next token.)

It is probably a silly example, but it illustrates how a macro can be overloaded based on its lexical context. Much of the expressive power of LuaMacro comes from allowing macros to fetch their own parameters in this way. It allows us to define new syntax and go beyond 'pseudo-functions', which is more important for a conventional-syntax language like Lua than Lisp (where everything looks like a function.)

It is entirely possible for macros to create macros; that is what `def_` does. Consider how to add the concept of `const` declarations to Lua:

    const N,M = 10,20

Here is one solution:

    macro.define ('const',function(get)
        get() -- skip the space
        local vars,values = get:names '=',get:list '\n'
        for i,name in ipairs(vars) do
            macro.assert(values[i],'each constant must be assigned!')
            macro.set_scoped_macro(name,tostring(values[i]))
        end
    end)

The key to making these constants well-behaved is `set_scoped_macro`, which installs a block handler which resets the macro to its original value, which is usually `nil`. This test script shows how the scoping works:

    require_ 'const'
    do
      const N,M = 10,20
      do
         const N = 5
         assert(N == 5)
      end
      assert(N == 10 and M == 20)
    end
    assert(N == nil and M == nil)


If we were designing a DSL intended for non-technical users, then we cannot just say to them 'learn the language properly - go read PiL!'. It would be easier to explain:

    forall x in {10,20,30} do

than the equivalent generic `for` loop. `forall` can be implemented fairly simply as a macro:

    macro.define('forall',function(get)
      local var = get:name()
      local t,v = get:next() -- will be 'in'
      local rest = tostring(get:upto 'do')
      return ('for _,%s in ipairs(%s) do'):format(var,rest)
    end)

That is, first get the loop variable, skip `in`, grab everything up to `do` and output the corresponding `for` statement.

Useful macros can often be built using these new forms. For instance, here is a simple list comprehension macro:

    macro.define('L(expr,select) '..
        '(function() local res = {} '..
        '  forall select do res[#res+1] = expr end '..
        'return res end)()'
    )

For example, `L(x^2,x in t)` will make a list of the squares of all elements in `t`.

(`macro.forall` defines more sophisticated `forall` statements and list comprehension expressions, but the principle is the same.)

There is a second argument passed to the substitution function, which is a 'putter' object - an object for building token lists. For example, a useful shortcut for anonymous functions:

    M.define ('\\',function(get,put)
        local args, body = get:names('('), get:list()
        return put:keyword 'function' '(' : names(args) ')' :
            keyword 'return' : list(body) : space() : keyword 'end'
    end)

The `put` object has methods for appending particular kinds of tokens, such as keywords and strings, and is also callable for operator tokens. These always return the object itself, so the output can be built up with chaining.

Consider `\x,y(x+y)`: the `names` getter grabs a comma-separated list of names upto the given token; the `list` getter grabs a general argument list. It returns a list of token lists and by default stops at ')'.  This 'lambda' notation was suggested by LHF as something easily parsed by any token-filtering approach - an alternative notation `|x,y| x+y` has been [suggested](http://lua-users.org/lists/lua-l/2009-12/msg00071.html) but is generally impossible to implement using a lexical scanner, since it would have to parse the function body as an expression. The `\` macro also has the advantage that the operator precedence is explicit: in the case of `\(42,'answer')` it is immediately clear that this is a function of no arguments which returns two values.

(Although I would not necessarily suggest that lambdas are a good thing in production code, they can be useful in iteractive exploration and within tests.)

Macros with explicit parameters can define a substitution function, but this function receives the values themselves, not the getter and putter objects. These values are _token lists_ and must be converted into the expected types using the token list methods:

    macro.define('test_(var,start,finish)',function(var,start,finish)
        var,start,finish = var:iden(),start:number(),finish:number()
        print(var,start,finish)
    end)


Since no `put` object is received, such macros need to construct their own:

        local put = M.Putter()
        ...
        return put

(They can of course still just return the substitution as text.)

### Dynamically controlling macro expansion

Consider this oop-unrolling macro:

    do_(i,1,3,
       y = y + 1
    )

which will expand as

    y = y + 1
    y = y + 2
    y = y + 3

For each iteration, it needs to define a local macro `i` which expands to 1,2 and 3.

    macro.define('do_(v,s,f,stat)',function(var,start,finish,statements)
        local put = macro.Putter()
        var,start,finish = var:iden(),start:number(),finish:number()
        macro.push_token_stack('do_',var)
        for i = start, finish do
            -- output `set_ <var> <value> `
            put:name 'set_':name(var):number(i):space()
            put:tokens(statements)
        end
        -- output `undef_ <var> <value>`
        put:name 'undef_':name(var)
        -- output `_POP_ 'do_'`
        put:name '_DROP_':string 'do_'
        return put
    end)

Ignoring the macro stack manipulation for a moment, it works by inserting `set_` macro assignments into the output. That is, the raw output looks like this:

    set_ i 1
    y = y + i
    set_ i 2
    y = y + i
    set_ i 2
    y = y + i
    undef_ i
    _DROP_ 'do_'

It's important here to understand that LuaMacro does not do _recursive_ substitution. Rather, the output of macros is pushed out to the stream which is then further substituted, etc. So we do need these little helper macros to set the loop variable at each point.

Using the macro stack allows macros to be aware that they are expanding inside a `do_` macro invocation.  Consider `tuple`, which is another macro which creates macros:

    tuple(3) A,B
    A = B

which would expand as

    local A_1,A_2,A_3,B_1,B_2,B_3
    A_1,A_2,A_3 = B_1,B_2,B_3

But we would like

    do_(i,1,3,
      A = B/2
    )

to expand as

    A_1 = B_1/2
    A_2 = B_2/2
    A_2 = B_2/2

And here is the definition:

    macro.define('tuple',function(get)
        get:expecting '('
        local N = get:number()
        get:expecting ')'
        get:expecting 'space'
        local names = get:names '\n'
        for _,name in ipairs(names) do
            macro.define(name,function(get,put)
                local loop_var = macro.value_of_macro_stack 'do_'
                if loop_var then
                    local loop_idx = tonumber(macro.get_macro_value(loop_var))
                    return put:name (name..'_'..loop_idx)
                else
                    local out = {}
                    for i = 1,N do
                        out[i] = name..'_'..i
                    end
                    return put:names(out)
                end
            end)
        end
    end)

The first expansion case happens if we are not within a `do_` macro; a simple list of names is outputted.  Otherwise, we know what the loop variable is, and can directly ask for its value.


### Implementation

It is not usually necessary to understand the underlying representation of token lists, but I present it here as a guide to understanding the code.

The token list representation of the expression `x+1` is:

    {{'iden','x'},{'+','+'},{'number','1'}}

which is the form returned by the LPeg lexical analyser. Please note that there are also 'space' and 'comment' tokens in the stream, which is a big difference from the token-filter standard.

The `TokenList` type defines `__tostring` and some helper methods for these lists.

The following macro is an example of the lower-level coding needed without the usual helpers:

    local macro = require 'macro'
    macro.define('qw',function(get,put)
      local append = table.insert
      local t,v = get()
      local res = {{'{','{'}}
      t,v = get:next()
      while t ~= ')' do
        if t ~= ',' then
          append(res,{'string','"'..v..'"'})
          append(res,{',',','})
        end
        t,v = get:next()
      end
      append(res,{'}','}'})
      return res
    end)

We're just using the getter `next` method to skip any irritating whitespace, but building up the substitution without a putter, just manipulating the raw token list.  `qw` takes a plain list of words, separated by spaces (and maybe commas) and makes it into a list of strings. That is,

    qw(one two three)

becomes

    {'one','two','three'}


The main loop of `macro.substitute` (towards end of `macro.lua`) summarizes the operation of LuaMacro:

There are two macro tables, `imacro` for classic name macros, and `smacro` for operator style macros. They contain macro tables, which must have a `subst` field containing the substitution and may have a `parms` field, which means that they must be followed by their arguments in parentheses.

A keywords table is chiefly used to track block scope, e.g. `do`,`if`,`function`,etc means 'increase block level' and `end`,`until` means 'decrease block level'. At this point, any defined block handlers for this level will be evaluated and removed. These may insert tokens into the stream, like macros. This is how something like `_END_CLOSE_` is implemented: the `end` causes the block level to decrease, which fires a block handler which passes `end` through and inserts a closing `)`.

Any keyword may also have an associated keyword handler, which works rather like a macro substitution, except that the keyword itself is always passed through first. (Allowing keywords as regular macros would generally be a bad idea because of the recursive substitution problem.)

The macro `subst` field may be a token list or a function. if it is a function then that function is called, with the parameters as token lists if the macro defined formal parameters, or with getter and setter objects if not. If the result is text then it is parsed into a token list.
