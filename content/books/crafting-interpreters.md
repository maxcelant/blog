---
title: Crafting Interpreters
---

### Chapter 4: Scanner
Scans a character at a time, and figures out what kind of _token_ each _lexeme_ is. 
#### Token Types
- **Keywords** - Reserved works like `while` or `fun`.
- **Identifiers** - Variable names like `foo`.
- **Single Character Tokens** - Things like `;` or `(`
- **Two Character Tokens** - Things like `==` or `!=`

#### Token Item
- An object that represents a complete token
```python
token = {
  token_type: TokenType.NUMBER,
  lexeme: '1',
  literal: 1.0,
  line: 15
}
```

### Chapter 5: Representing Code
#### Expression Problem
- It's difficult to add new functionality to a group of related types.
- In a OO language, that would involve adding a new method to ALL of the types.
- For instance a set of `Car` types with `honk`, and `drive` methods. If you wanted to add `drift` method, you would have to add it to each type individually.
- Visitor design pattern fixes this.

#### Visitor Pattern
- Allows you to have a single `accept(visitor: Visitor)` method in each type (`Car`) simply `visits` the intended implementation of the Visitor.
- So in the example above, we could create concrete visitors for `VisitDrive`, `VisitHonk`, `VisitDrift`.
- This keeps the implementation details of a method separate from the class itself.

```ts
interface Visitor {
  visitRaceCar(raceCar: RaceCar);
  visitTruck(truck: Truck)
}

abstract class Car {
  abstract accept(visitor: Visitor);
}

class RaceCar extends Car {
  accept(visitor: Visitor) {
    visitor.visitRaceCar(this);   
  }
}

class Truck extends Car {
  accept(visitor: Visitor) {
    visitor.visitTruck(this);   
  }
}

class DriftVisitor implements Visitor {
  visitRaceCar(raceCar: RaceCar) {
    // do something
  }

  visitTruck(truck: Truck) {
    // do something
  }
}
```

### Context Free Grammars

In the syntactic grammar we’re talking about now, we’re at a different level of granularity. Now each “letter” in the alphabet is an entire token and a “string” is a sequence of tokens—an entire expression.

| Terminology            | Lexical Grammar | Syntactic Grammar |
| ---------------------- | --------------- | ----------------- |
| The "alphabet" is...   | Characters      | Tokens            |
| A "string" is...       | Lexeme or token | Expression        |
| It's implemented by... | Scanner         | Parser            | 

A formal grammar’s job is to specify which strings are valid and which aren’t. If we were defining a grammar for English sentences, “eggs are tasty for breakfast” would be in the grammar, but “tasty breakfast for are eggs” would probably not.

#### Example

- **Terminal** - Literal values, like `"with"` or `"on the side"` in the example below.
- **Non-Terminal** - Reference to another rule in the grammar. Like `protein` or `cooked`.
- **Rule** - Is on the left side like `breakfast`, `protein`, etc.

From this context-free grammar, we can create different expressions!

```js
breakfast → protein "with" breakfast "on the side" ;
breakfast → protein ;
breakfast → bread ;

protein → crispiness "crispy" "bacon" ;
protein → "sausage" ;
protein → cooked "eggs" ;

crispiness → "really" ;
crispiness → "really" crispiness ;

cooked → "scrambled" ;
cooked → "poached" ;
cooked → "fried" ;

bread → "toast" ;
bread → "biscuits" ;
bread → "English muffin" ;
```

```js
protein "with" breakfast "on the side"
crispiness "crispy" "bacon" "with" breakfast "on the side"
"really" "cripsy" "bacon" "with" breakfast "on the side"
"really" "cripsy" "bacon" "with" bread "on the side"
"really" "cripsy" "bacon" "with" "toast" "on the side"
```

Notice how we replace the non terminal with one of its possible outputs, that output may be a terminal or non-terminal.

#### Enhancing the Notation

Use `|` to represent each possible choice.

```js
bread → "toast" | "biscuits" | "english muffin" ; 
```

Use parenthesis for grouping and `|` for selecting on of the options

```js
protein → ("scrambled" | "poached" | "fried" ) "eggs" ; 
```

`*` postfix means you can repeat the previous symbol 0 or more times

```js
crispiness → "really" "really"* ; 
```

`+` postfix means it must appear 1 or more times.

```js
crispiness → "really"+ ; 
```

`?` postfix is for an optional production. It can appear 0 or 1 times.

```js
breakfast → protein ( "with" breakfast "on the side" )? ;
```

Full enhanced context:

```js
breakfast → protein ( "with" breakfast "on the side" )? | bread

protein → "really"+ "crispy" "bacon" 
          | "sausage" 
          | ("scrambled" | "poached" | "fried" ) "eggs"

bread → "toast" | "biscuits" | "english muffin";
```

##### Grammar For Lox Language

```js
expression → literal | unary | binary | grouping ;

literal → NUMBER | STRING | "true" | "false" | "nil" ;

grouping → "(" expression ")" ;

unary → ( "-" | "!" ) expression ;

binary → expression operator expression ;

operator → "==" | "!=" | "<" | "<=" | ">" |
		   ">=" |"+" |"-" | "*" | "/";
```

`CAPITALIZED` terminals represent number or string literals.

### Chapter 6: Parsing Expressions
#### Precedence

We need to define a different rule for each precedence, otherwise there will be no order of computation in our parse tree!

**Completed Expression Grammar**
```python
expression → equality ;

equality → comparison ( ( "!=" | "==" ) comparison )* ;

comparison → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;

term → factor ( ( "-" | "+" ) factor )* ;

factor → unary ( ( "/" | "*" ) unary )* ;

unary → ( "!" | "-" ) unary | primary ;

primary → NUMBER | STRING | "true" | "false" | "nil"
| "(" expression ")" ;
```

```ad-note
In this about notation, `primary` has the highest precedence meaning that numbers, strings, booleans and parenthesized expressions are evaluted before anything else.
```

An important note is that each rule needs to match expressions at that precedence level _or higher_, so we need to let this match a primary expression.

```python
unary → ( "!" | "-" ) unary | primary ;
```


```ad-example
**RULE:** `factor → unary ( ( "/" | "*" ) unary )* ;`
1. `unary / unary * unary`
2. `-primary / primary * primary`
3. `-5 / 3 * 10`
```

##### Recursive Descent

One of my parsing techniques. Considered a "top-down parser" because it starts from the outermost grammar (`expression`) and goes down to the nested subexpressions.

```ad-tldr
The descent is described as “recursive” because when a grammar rule refers to itself -- directly or indirectly--that translates to a recursive function call.
```

**Rule Translation**

| Grammar     | Code                              |
| ----------- | --------------------------------- |
| Terminal    | Code to match and consume a token |
| Nonterminal | Call to that rule's function      |
| "\|"        | `if` or `switch` statement        |
| * or +      | `while` or `for` loop             |
| "?"         | `if` statement                    |

The reason the lower precedence call the higher precedence terms is because those will be calculated first in the parse tree! Aka they will be the leaf nodes that we compute first.

```python
  def equality(self) -> Expr:
    expr: Expr = self.comparison()

    while self.match(TokenType.BANG_EQUAL, TokenType.EQUAL_EQUAL):
      operator: TokenItem = self.previous()
      right: Expr = self.comparison()
      expr = Expr.Binary(expr, operator, right)

    return expr
```

Because of the recursive nature, the value of `expr` evolves from just being a value to being the entire left side expression and so one.

##### Parser Class

```python
class Parser:
  def __init__(self, tokens: list[TokenItem]):
    self.tokens  = tokens
    self.current = 0

  def match(self, *types: TokenType) -> bool:
    for t in types:
      if self.check(t):
        self.advance()
        return True
    return False
  
  def check(self, token_type: TokenType) -> bool:
    if self.is_at_end():
      return False
    return self.peek().token_type == token_type

  def advance(self) -> TokenItem:
    if not self.is_at_end():
      self.current += 1
    return self.previous()
  
  def is_at_end(self) -> bool:
    return self.peek().token_type == TokenType.EOF
  
  def peek(self) -> TokenItem:
    return self.tokens[self.current]
  
  def previous(self) -> TokenItem:
    return self.tokens[self.current - 1]
```

- `match`: looks through types and sees if the current token (we haven't consumed yet) is of that type. If so, it consumes it and returns true.
- `check`: sees if `token_type` of the token is equal to the given parameter `token_type`
- `advance`: Moves the `current` pointer up 1 and returns the previous token.
- `is_at_end`: checks to see if we've reached the last token in the list.
- `peek`: Looks at current token _we have yet to consume_.
- `previous`: Looks at the previous token (most recently consumed).

##### Synchronizing and Panic Mode

When the parser finds something that doesn't make sense, it enters "panic mode" to handle the mistake.

In _our_ case, we know we reach the end of statement because we reach a semicolon. Which gives us a great indicator for synchronization.

1. **Find Synchronization Point:** The parser looks for a specific point in the code that it understands (a "synchronization point"), which is usually a place where a new statement or a clear structure begins.
2. **Jump to Synchronization Point:** It then skips or ignores parts of the code until it reaches this point. This way, it aligns itself with a part of the code that makes sense again.
3. **Resume Parsing:** With the parser and the code aligned at a recognizable point, parsing can continue, hopefully without being thrown off by the initial error.

```python
  def synchronize(self) -> None:
    self.advance()

    while not self.is_at_end():
      if self.previous().token_type == TokenType.SEMICOLON:
        return

      if self.peek().token_type in {
        TokenType.CLASS,
        TokenType.FUN,
        TokenType.VAR,
        TokenType.FOR,
        TokenType.IF,
        TokenType.WHILE,
        TokenType.PRINT,
        TokenType.RETURN,
      }:
        return
      
      self.advance()
```

### Chapter 7: Evaluating Expressions
##### Expressions
These are our terms for the language (as it stands).

```python
class Expr(ABC):
  @abstractmethod
  def accept(self, visitor: Visitor):
    pass


  class Binary:
    def __init__(self, left: Expr, operator: TokenItem, right: Expr):
      self.left = left
      self.operator = operator
      self.right = right

    def accept(self, visitor: Visitor):
      return visitor.visit_binary(self)


  class Grouping:
    def __init__(self, expression: Expr):
      self.expression = expression

    def accept(self, visitor: Visitor):
      return visitor.visit_grouping(self)


  class Literal:
    def __init__(self, value: object):
      self.value = value

    def accept(self, visitor: Visitor):
      return visitor.visit_literal(self)


  class Unary:
    def __init__(self, operator: TokenItem, right: Expr):
      self.operator = operator
      self.right = right

    def accept(self, visitor: Visitor):
      return visitor.visit_unary(self)
```

### Chapter 8: Statements and State
### Expressions vs Statements
##### Expressions:
Expressions evaluate to a value. They can be simple calculations, function calls that return a value, or any other code that results in a value.

1. `42` - A simple numeric value.
2. `"Hello, World!"` - A string literal is also an expression.
3. `3.14 * r * r` - Assuming `r` is defined, this expression calculates the area of a circle.
4. `[1, 2, 3, 4].pop()` - Calls the `pop` method, which is an expression that returns a value (the last item of the list).
5. `len("Python")` - The `len` function call is an expression that returns the length of the string `"Python"`.

##### Statements:
Statements perform an action. They might include variable assignments, loops, conditionals, etc., and don’t necessarily have to produce a value.

1. `import sys` - An import statement that brings a module into the current namespace.
2. `x = 5` - An assignment statement that assigns the value `5` to the variable `x`.
3. `print("Hello, World!")` - A print statement that outputs a string to the console.
4. `for i in range(5): print(i)` - A loop statement that iterates and prints numbers 0 through 4.
5. `if x > 0: print("Positive")` - A conditional statement that checks if `x` is greater than 0 and then prints "Positive".

**Differences Recap**:
- Expressions are parts of code that calculate a value.
- Statements are entire lines of code that perform actions and don’t necessarily produce a value directly.
- While statements represent an action or command, expressions are involved in producing values. Statements might contain expressions, but not the other way around.

---
### Statement Grammar

```c
program → statement* EOF ;

declaration → varDecl | statement ;

statement → exprStmt | printStmt ;

varDecl → "var" IDENTIFIER ( "=" expression )? ";" ;

exprStmt → expression ";" ;

printStmt → "print" expression ";" ;
```

---
### Stmt Class

The `Stmt` class kind of acts as a wrapper for the `Expr` class. Since [[Statement Grammar]] are _actions_, they wrap over the expression.

```ad-tip
We _execute_ as a "statement".
We _evaluate_ a "expression".
```

```python
class Stmt(ABC):

  @abstractmethod
  def accept(self, visitor: Stmt.Visitor):
    pass


  class Visitor(ABC):
    @abstractmethod
    def visit_print_stmt(self, expr: Stmt.Expr):
      pass


    @abstractmethod
    def visit_expression_stmt(self, expr: Stmt.Expr):
      pass


  class Expression:
    def __init__(self, expression: Expr):
      self.expression = expression


    def accept(self, visitor: Stmt.Visitor):
      visitor.visit_expression_stmt(self)


  class Print:
    def __init__(self, expression: Expr):
      self.expression = expression


    def accept(self, visitor: Stmt.Visitor):
      visitor.visit_print_stmt(self)

```

The `Stmt` class also has its own Visitor where it can handle those specific calls like printing or just performing a normal expression. More will be added later.

---
### l-values and r-values
- When the parser sees an assignment operation (like `a = "value";`), it doesn't know it's an assignment until it encounters the = sign. This is because, before seeing the =, it could be any kind of expression.
- **L-values and R-values:** The key point is understanding the difference between an expression that evaluates to a value (`r-value`) and something that identifies a location where a value can be stored (`l-value`).
	- **R-value:** An expression that produces a value. For example, in `5 + 3`, both `5` and `3` are r-values, and the expression itself evaluates to another r-value, `8`.
	- **L-value:** An expression that identifies a storage location. For example, in `a = 5`, `a` is an l-value because it represents a location where the value `5` can be stored.
- **Why It Matters:** The parser needs to treat l-values and r-values differently. For an assignment, the left-hand side must be an l-value because you're assigning a value to a location. However, the parser won't know it's dealing with an l-value until it encounters the assignment operator =. This makes parsing assignments more complex than other expressions.

---
### Assignment Grammar

```js
expression → assignment ;

assignment → IDENTIFIER "=" assignment | equality ; // NEW!

equality → comparison ( ( "!=" | "==" ) comparison )* ;
```
##### Assignment Code Block

```java
private Expr assignment() {
    Expr expr = equality();
    if (match(EQUAL)) {
      Token equals = previous();
      Expr value = assignment();
      if (expr instanceof Expr.Variable) {
         Token name = ((Expr.Variable)expr).name;
         return new Expr.Assign(name, value);
	  }
      error(equals, "Invalid assignment target.");
    }
    return expr;
  }
```

This piece of code is part of a parser in a programming language interpreter, specifically designed to handle assignment expressions like `a = 5;`. Here's a simplified explanation:

1. **Start with an expression**: The method begins by evaluating an expression to the left of the assignment operator (=) using `equality()`. This could be a variable or any expression that boils down to a value.
2. **Check for assignment**: It checks if the next token is an assignment operator (=) using `match(EQUAL)`. If it's not, the method just returns the expression evaluated in step 1.
3. **Recursive assignment handling**: If there is an assignment operator, the method recursively calls `assignment()` again to evaluate the right-hand side of the assignment. This allows for handling of nested assignments like `a = b = 5;`.
4. **Handle variable assignments**: If the left-hand side of the assignment is a variable (`expr instanceof Expr.Variable`), it extracts the variable's name and constructs an `Expr.Assign` object. This object represents the assignment operation, holding both the variable's name and the value to be assigned to it.
5. **Error handling**: If the left-hand side is not a variable (for example, an attempt to assign a value to a number or a direct value like `5 = a;`), it triggers an error reporting method with a message saying "Invalid assignment target."

---
#### Block Scope

- **Lexical scope (or static scope)** is a style of scope where the program shows where the scope starts and ends.
- In Lox, curly-braces control scope. 

##### Simple implementation of Block Scope

1. As we visit statements in a block, keep track of variables declared.
2. When the last statement is executed, tell the environment to delete all those variables.

##### Environment Chaining
- We define a fresh environment for each block containing only variables defined in that scope.
- We need to chain environments together. Because inner scopes should still have reference to variables in outer scopes.
- Start with innermost scope and go out until we find the variable. 

> ![[Pasted image 20240407212405.png]]

##### Environment Class Update

- Each environment will have a reference to the scope that wraps it. So checking the `enclosing` is like going up!

```python
class Environment:
  def __init__(self, enclosing: Environment | None = None):
    self.values = {}
    self.enclosing = enclosing            # Nested environment

  def get(self, name: TokenItem):
    if name.lexeme in self.values:
      return self.values[name.lexeme]
    if self.enclosing:
      return self.enclosing.get(name)     # Check nested environment
    
    raise RuntimeException(name, f"...")
```

##### Block Syntax and Semantics

We are going to add to the [[Statement Grammar]].

```c
declaration → varDecl | statement ;

statement → exprStmt | printStmt | block ;

block → "{" declaration* "}"
```


##### Parser Code

```python
def statement(self) -> Stmt:
  if self.match(TokenType.IF):
    return self.if_statement()
  if self.match(TokenType.PRINT):
    return self.print_statement()
  if self.match(TokenType.LEFT_BRACE):
    return Stmt.Block(self.block()) # <---
  return self.expression_statement()

...

def block(self) -> list[Stmt]:
  statements: list[Stmt] = []

   while not self.check(TokenType.RIGHT_BRACE) and \
     not self.is_at_end():
      statements.append(self.declaration())

  self.consume(TokenType.RIGHT_BRACE, "Expect '}' after block.")
  return statements
```

If we see a left bracket, start funneling all the statements into a list of statements until we see a closing right bracket.

##### Interpreter Code

```python
  def execute_block(self, statements, environment):
    previous = self.environment
    try:
      self.environment = environment
      for stmt in statements:
        self.execute(stmt)
    finally:
      self.environment = previous
```

When executing a block of code, the environment represents the _current_ one. Then it resets once the block is done, getting rid of those variables.

We just loop through those statements that we created in the parser and execute them.

### Chapter 9: Control Flow
### If Statements

```c
statement → exprStmt
			| ifStmt
			| printStmt
			| block ;
			
ifStmt → "if" "(" expression ")" statement
		  ( "else" statement )? ;
```
##### Dangling Else Problem

> ![[Pasted image 20240409120204.png]]

We solve this by having the `else` be a part of the closest `if` block.

##### Parser Code

```python
def if_statement(self) -> Stmt:
    self.consume(TokenType.LEFT_PAREN, "Expect '(' after 'if'.")
    condition: Expr = self.expression()
    self.consume(TokenType.RIGHT_PAREN, "Expect ')' after if condition.")

    then_branch: Stmt = self.statement()
    else_branch: Stmt | None = self.statement() if self.match(TokenType.ELSE) else None
    
    return Stmt.If(condition, then_branch, else_branch) 
```

Get the condition, then get the then and _possibly_ else blocks.

##### Interpreter Code

```python
  def visit_if_stmt(self, stmt: Stmt.If) -> None:
    if self.is_truthy(self.evaluate(stmt.condition)):
      self.execute(stmt.then_branch)
    elif stmt.else_branch:
      self.execute(stmt.else_branch)
```

Pretty straight forward, evaluate the condition and execute the correct path.

---
### Logical Operators

```c
expression → assignment ;

assignment → IDENTIFIER "=" assignment | logic_or ;

logic_or → logic_and ( "or" logic_and )* ;

logic_and → equality ( "and" equality )* ;
```

As you can see, they are pretty low on the precedence table.

##### Parser Code

```python
  def logic_or(self) -> Expr:
    expr: Expr = self.logic_and()

    while self.match(TokenType.OR):
      operator: TokenType = self.previous()
      right: Expr = self.logic_and()
      expr = Expr.Logical(expr, operator, right)

    return expr
  

  def logic_and(self) -> Expr:
    expr: Expr = self.equality()

    while self.match(TokenType.AND):
      operator: TokenType = self.previous()
      right: Expr = self.equality()
      expr = Expr.Logical(expr, operator, right)

    return expr
```

Following the grammar, this makes sense. Check to see if the tokens create an `or` or `and` expression.

##### Interpreter Code

```python
  def visit_logical_expr(self, expr: Expr.Logical):
    left: object = self.evaluate(expr.left)

    if expr.operator.type == TokenType.OR:
      if self.is_truthy(left): 
        return left
    else:
    # If its an AND operation and the left side is  
    # False, then theres no point checking the right
      if not self.is_truthy(left): 
        return left
    
    return self.evaluate(expr.right)
```

If the left hand side resolves to a _truthy_ value, then we dont even look at the right side, we just return the left.

This is exactly what python does.

---
### While Loops

##### Grammar

```c
statement → exprStmt | ifStmt | printStmt | whileStmt | block ;

whileStmt → "while" "(" expression ")" statement ;
```

##### Parser

Very straight forward. Consume the first parenthesis. The thing that follows is the condition, which is consumed as an expression.

The body is a statement, which means it could be pretty much anything, but most likely a `stmt.Body`.

```python
  def while_statement(self) -> Stmt:
    self.consume(TokenType.LEFT_PAREN, "Expect '(' after 'while'.")
    condition: Expr = self.expression()
    self.consume(TokenType.RIGHT_PAREN, "Expect ')' after condition.")
    body: Stmt = self.statement()

    return Stmt.While(condition, body)
```

##### Interpreter

We simply loop while the condition is true and execute the body on each iteration.

```python
  def visit_while_stmt(self, stmt: Stmt.While) -> None:
    while self.is_truthy(self.evaluate(stmt.condition)):
      self.execute(stmt.body)
```

---
#### For Loops
##### Grammar 

```c
statement → exprStmt
			| forStmt
			| ifStmt
			| printStmt
			| whileStmt
			| block ;

forStmt → "for" "(" ( varDecl | exprStmt | ";" )
			  expression? ";"
			  expression? ")" statement ;

```

For loop is really just _syntactic sugar_ on top of a while loop

1. First portion finds the initializer which could be nothing, a declaration or an expression.
2. The condition is either an expression or nothing.
3. The increment is either an expression or nothing as well.
4. The body is created by seeing whats after the for parenthesis. Thats put in the `body` variable.
5. Next we build on top of that by adding the incrementer into that block!
6. Next we create a `While` loop by stuffing that body with a possible condition.
7. Next we stuff the initializer before the while loop even runs and put all of that in another block.

```python
def for_statement(self) -> Stmt:
    self.consume(TokenType.LEFT_PAREN, "Expect '(' after 'for'.")

	# 1
    initializer: Stmt
    if self.match(TokenType.SEMICOLON):
      initializer = None
    elif self.match(TokenType.VAR):
      initializer = self.var_declaration()
    else:
      initializer = self.expression_statement()

	# 2
    condition: Expr = None
    if not self.check(TokenType.SEMICOLON):
      condition = self.expression()
    self.consume(TokenType.SEMICOLON, "Expect ';' after loop condition.")

	# 3
    increment: Expr = None
    if not self.check(TokenType.RIGHT_PAREN):
      increment = self.expression()
    self.consume(TokenType.RIGHT_PAREN, "Expect ')' after for clauses.")

	# 4
    body: Stmt = self.statement()

	# 5
    if increment:
      body = Stmt.Block([body, Stmt.Expression(increment)])

	# 5
    if not condition:
      condition = Expr.Literal(True)
    body = Stmt.While(condition, body)

	# 7
    if initializer:
      body = Stmt.Block([initializer, body])

    return body
```

### Chapter 10: Functions
- In call syntax, the **"callee"** just needs to result to a callable function, it doesn't necessarily need to be a name like `average` in `average(1,2)`. It's more flexible. 
- The **"call syntax"** is the parenthesis `()` that follows the callee.

```js
getCallback()()
```

- In the example `getCallback` would be the callee for the first pair of parenthesis.
- `getCallback()` would be the callee for the second pair of parenthesis.

```go
expression     → assignment ;
assignment     → IDENTIFIER "=" assignment | logic_or ;
logic_or       → logic_and ( "or" logic_and )* ;
logic_and      → equality ( "and" equality )* ;
equality       → comparison ( ( "!=" | "==" ) comparison )* ;
comparison     → term ( ( ">" | ">=" | "<" | "<=" ) term )* ;
term           → factor ( ( "-" | "+" ) factor )* ;
factor         → unary ( ( "/" | "*" ) unary )* ;
unary          → ( "!" | "-" ) unary | call ;

call           → primary ( "(" arguments? ")" )* ; // ⭐️ NEW!
primary        → NUMBER | STRING | "true" | "false" | "nil" | IDENTIFIER | "(" expression ")" ;
arguments      → expression ( "," expression )* ; // ⭐️ NEW!
```

Notice how `primary` is all it needs to be, making the "callee" quite flexible
