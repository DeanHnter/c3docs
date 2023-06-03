# Compile time introspection and execution

During compilation, constant expressions will automatically be folded. Together with the compile
time conditional statements `$if`, `$switch` and the compile time iteration statments `$for` `$foreach`
it is possible to perform limited compile time execution.

### Compile time values

During compilation, global constants are considered compile time values, as are any 
derived constant values, such as type names and sizes, variable alignments etc.

Inside of a macro or a function, it is also possible to define mutable compile time variables. Such
local variables are prefixed with `$` (e.g. `$foo`). It is also possible to define local *type* variables,
that are also prefixed using `$` (e.g. `$MyType` `$ParamType`).

Mutable compile time variables are *not* allowed in the global scope.

### $if and $switch

`$if (<const expr>):` takes a compile time constant value and evaluates it to true or false.

    macro foo($x, $y)
    {
        $if ($x > 3):
            $y += $x * $x;
        $else
            $y += $x;
        $endif
    }
    
    const int FOO = 10;
    
    fn void test()
    {
        int a = 5;
        int b = 4;
        foo(1, a); // Allowed, expands to a += 1;
        // foo(b, a); // Error: b is not a compile time constant.
        foo(FOO, a); // Allowed, expands to a += FOO * FOO;
    }

For switching between multiple possibilities, use `$switch`.

    macro foo($x, $y)
    {
        $switch ($x)
            $case 1: 
                $y += $x * $x;
            $case 2:
                $y += $x;
            $case 3:
                $y *= $x;
            $default:
                $y -= $x;
        $endif
    }

Switching without argument is also allowed, which works like an if-else chain:

    macro foo($x, $y)
    {
        $switch 
            $case $x > 10: 
                $y += $x * $x;
            $case $x < 0:
                $y += $x;
            $default:
                $y -= $x;
        $endif
    }

### Loops using $foreach and $for

Unlike `$if` and `$switch`, iteration is only allowed inside of functions and macros. 

`$for` ... `$endfor` works analogous to `for`, only it is limited to using compile time variables. `$foreach` ... `$endforeach` similarly 
matches the behaviour of `foreach`.

Compile time looping:

    macro foo($a)
    {
        $for (var $x = 0; $x < $a; $x++)
            io::printfn("%d", $x);     
        $endfor
    }
    
    fn void test()
    {
        foo(2);
        // Expands to ->
        // io::printfn("%d", 0);     
        // io::printfn("%d", 1);         
    }

Looping over enums:

    macro foo_enum($SomeEnum)
    {
        $foreach ($x : $SomeEnum.values)
            io::printfn("%d", (int)$x);     
        $endforeach
    }
    
    enum MyEnum
    {
        A,
        B,
    }
    
    fn void test()
    {
        foo_enum(MyEnum);
        // Expands to ->
        // io::printfn("%d", (int)MyEnum.A);
        // io::printfn("%d", (int)MyEnum.B);    
    }

An important thing to note is that the content of the `$foreach` or `$for` body must be at least a complete statement.
It's not possible to compile partial statements.

### Compile time macro execution

If a macro only takes compile time parameters, that is only `$`-prefixed parameters, and then does not generate 
any other statements than returns, then the macro will be completely compile time executed.

    macro @test($abc)
    {
        return $abc * 2;
    }

    const int MY_CONST = @test(2); // Will fold to "4"

This constant evaluation allows us to write some limited compile time code. For example, this
macro will compute Fibonacci at compile time:

    macro long @fib(long $n)
    {
        $if ($n <= 1):
            return $n;
        $else
            return @fib($n - 1) + @fib($n - 2);
        $endif
    }

It is important to remember that if we had replaced `$n` with `n` the compiler would have complained. `n <= 1` 
is not be considered to be a constant expression, even if the actual argument to the macro was a constant.
This limitation is deliberate, to offer control over what is compiled out and what isn't.

### Conditional compilation at the top level

At the top level, conditional compilation is done with `$if` and `$switch`, just like the inside of 
macros and functions, but rather than statements, any *complete* declarations may be conditionally
included.

    $if ($defined(platform::OS) && platform::OS == WIN32)
    
    fn void doSomethingWin32Specific()
    {
        /* .... */
    }
    
    $endif

#### Evaluation order of top level conditional compilation

Conditional compilation at the top level can cause unexpected ordering issues, especially when combined with 
`$defined`. It is strongly recommended to only switch on unconditionally defined values.

## Compile time introspection

At compile time, full type information is available. This allows for creation of reusable, code generating, macros for things
like serialization.

To read more about all the fields available at compile time, see the page on [reflection](../reflection).

