# Debugger Operator

This is a draft proposal for the TC39 commitee. It proposes extending the `debugger`
keyword (which is currently a statement) to allow it to be used as
a unary operator. This would allow breakpoints to be placed mid-expression, and would
reduce the amount of syntax the user has to type to place a breakpoint in many scenarios.

## Motivation

Consider the following situations:

```

// example 1
fn(fn1(), fn2());

// example 2
if (fn1() && fn2()) {}

// example 3
fn1((a,b) => fn2(b,a));

// example 4
fn({a: fn1(), b: fn2()});

```

In each case, imagine you want to insert a breakpoint using `debugger` before
the call to `fn2` but after the call to `fn1`. This is needlessly teduous because
`DebuggerStatement` is a statement. The code would need to be changed like so:

```

// example 1
fn(fn1(), (() => {debugger;})() || fn2());
// note: can't do this - it's a syntax error
fn(fn1(), (() => debugger)() || fn2());

// example 2
if (fn1() && ((() => {debugger;})() || fn2())) {}

// example 3
fn1((a,b) => {debugger; return fn2(b,a)});

// example 4
fn({a: fn1(), b: (() => {debugger; return fn2()})});

```

This proposal adds a `debugger` unary operator (without removing `DebuggerStatement`).
This operator is a UnaryExpression, like the `typeof`, `void`, and `delete` operators.
It has precidence 14, above `delete`. **It triggers a breakpoint, then evaluates
it's input, then returns it's input.**

In each example, we can insert a breakpoint before `fn2` by literally writing "debugger"
before `fn2`:

```

// example 1
fn(fn1(), debugger fn2());

// example 2
if (fn1() && debugger fn2()) {}

// example 3
fn1((a,b) => debugger fn2(b,a));

// example 4
fn({a: fn1(), b: debugger fn2()});

```



## Why an Operator rather than an Expression?

Making `debugger` an expression instead of an operator enables `() => debugger`,
but has drawbacks.

### Return value would be meaningless
If `debugger` is an expression, it must return something. The behavior of one of
the below cases is affected by the addition of the `debugger` expression, and
the other is not. It's not clear which one is affected because the return
value of `debugger` has no semantic meaning.

```
if (a && b && debugger && c && d) {}
if (a || b || debugger || c || d) {}
```

If `debugger` evaluates to a truthy value, then the bottom statement would
have different behavior with the debugger expression. Else, the top statement
would have different behavior.

### Awkward syntax required to avoid altering existing behavior

If `debugger` were an expression, adding it to the front of an arrow
function body without altering the function's behavior requires awkward syntax.

```
// Current behavior:
// Works:
fn(() => {debugger;});
// SyntaxError
fn(() => debugger);

// If debugger were an expression
// Works
fn(() => debugger);
// Works, but requires brackets and comma operator
fn(() => (debugger,fn2()));

// If debugger were an operator
// Still works (this proposal doesn't remove DebuggerStatement)
fn(() => {debugger;});
// Still a SyntaxError
fn(() => debugger);
// Works
fn(() => debugger 0);
fn(() => debugger fn2());

```

### Potential for confusion

If `debugger` were an expression, then `if (debugger) {}` would be valid syntax (as would
many other nonsence constructs).

Someone who does not know what the debugger keyword does may assume that `if (debugger) {}`
is a method for checking if the application is running in debug mode or for checking if a
debugger is attached.


## Why not a postfix version

A postfix debugger operator (i.e. ) to allow creation of breakpoints after
a statement has been evaluated could also be useful. In the below example, the breakpoint
would be triggered after the `fn2` call but before the `fn1` call.

```
fn1(fn2() debugger);
```

This would not be worthwhile, because most debug tools have step in/out/over functions.
Therefore it would be easy to put the breakpoint before the `fn2` call, then step past
to `fn2` call to get the desired result.

Additionally, pre/post debugger operators may be more difficult to implement. There are
already 3 `UnaryExpression`s, adding a fourth should not be unduely difficult.


## Display in debugging tools

If a breakpoint is placed midway through an expression, then debuggers should render that
differently than if the breakpoint is placed on the line. To test how a debugger would
render the result of a breakpoint in the middle of an expression, we can use
`0 || (() => {debugger})() || 1`. The results follow:

 * Edge Debugger - Highlights only the debugger statement, and draws an arrow in
    the gutter of the affected line. The cursor is moved to the start of the
    `debugger` statement.
 * Chrome Inspector - Highlights from the breakpoint location to the end of the
    line (a bug?). Also highlights the entire line in a lighter color. The
    cursor is moved to the start of the `debugger` statement.
 * FireFox Debugger - Highlights the entire line. The cursor is placed on the first
    column of the line.
 * WebStorm Debugger - Highlights the entire line. The cursor is placed on the first
    non-whitespace column of the line.
 * VS Code Debugger - Highlights the entire line. The cursor is placed on the first
    column of the line.
 * Eclipse Debugger - Highlights the entire line. The cursor is placed on the first
    column of the line.

While not essential, a more detailed rendering in the debugger would be desirable as the
behavior of `debugger` becomes more intricate. For the operator version of `debugger`, it
may be best to place the cursor on the start of the expression after the `debugger`
operator.

## Changes to `DebuggerStatement`

In the current spec, `DebuggerStatement` returns an implementation defined completion
value if a debugger is attached. In order to behave consistently with the `debugger`
operator, `DebuggerStatement` should return `undefined`.

No implementations seem to return anything other than `undefined`. Regardless, this
change would not be significant because the only way to access `DebuggerStatement`'s
return value is with `eval('debugger;');` or equivalent.






## Specification


### Unary Operators (replace section 12.5)

#### Syntax

*UnaryExpression*<sub>[Yield]</sub>: <br/>
&nbsp;&nbsp;&nbsp;&nbsp;UpdateExpression*<sub>[Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`debugger` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`delete` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`void` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`typeof` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`+` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`-` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`~` UnaryExpression*<sub>[?Yield]</sub> <br/>
&nbsp;&nbsp;&nbsp;&nbsp;`!` UnaryExpression*<sub>[?Yield]</sub> <br/>

#### Static Semantics: IsFunctionDefinition

*UnaryExpression:* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*UpdateExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`debugger` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`delete` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`void` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`typeof` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`+` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`-` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`~` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`!` UnaryExpression* <br/>

*Return `false`.*

#### Static Semantics: IsValidSimpleAssignmentTarget

*UnaryExpression:* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*UpdateExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`debugger` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`delete` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`void` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`typeof` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`+` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`-` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`~` UnaryExpression* <br/>
&nbsp;&nbsp;&nbsp;&nbsp;*`!` UnaryExpression* <br/>

*Return `false`.*

### The `debugger` operator (new section before 12.5.3 (The delete operator))

*UnaryExpression : `debugger` UnaryExpression* <br/>

#### Runtime Semantics: Evaluation

*UnaryExpression : `debugger` UnaryExpression*

If an implementation defined debugging facility is available and enabled, then <br/>
&nbsp;&nbsp;&nbsp;&nbsp;Perform an implementation defined debugging action. <br/>
Let `expr` be the result of evaluating UnaryExpression. <br/>
Let `value` be the result of `? GetValue(expr)`. <br/>
Return `value`. <br/>

### The `debugger` Statement (replace section 13.16)

#### Syntax

*DebuggerStatement : `debugger`;* <br/>
    
#### Runtime Semantics: Evaluation

*DebuggerStatement : `debugger` ;* <br/>

If an implementation defined debugging facility is available and enabled, then <br/>
&nbsp;&nbsp;&nbsp;&nbsp;Perform an implementation defined debugging action. <br/>
Let `result` be NormalCompletion(empty). <br/>
Return `result`. <br/>

