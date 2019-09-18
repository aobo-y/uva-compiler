# HW1

## Part 1

### 2

#### (a)

Three distinct cases of type analysis are discussed in this section.

The first case is `AProgram`. Below is the corresponding code snippet.

```scala
case program: AProgram =>
  if (program.hasMainFunction) {
    val mf = program.mainFunction
    for (arg <- mf.args)
      unify(arg, TipInt())

    unify(mf.stmts.ret.value, TipInt())
  }
```

According to TIP's specification, the parameters and the return value of the `main` function of a TIP program must be `int`. The constraint is
```
main(X1, ..., Xn){ ... return E; }: [X1] = ... [Xn] = [E] = int
```
This code snippet will check if a program AST node has `main`. If it does, each argument and the return value of the `main` will be unified with type integer. It is unnecessary to add contraint of type `(int, ..., int) -> int` to the `main` function, because the function type contraint will be added when the `main` function node is visited as the children of the program node.

The second case is `AAssignStmt`. Its implemenation is shown below.

```scala
case ass: AAssignStmt =>
  unify(ass.left, ass.right)
```

It is quite straightforward that the operands on the two sides of the assign operator (`=`) in an assigning statement must belong to the same type. The constraint of assignment is `X == E: [X] = [E]`. Hence, the code simply unify them.

The third case is `AFunDeclaration`. The corresponding code is shown below.

```scala
case fun: AFunDeclaration =>
  unify(fun, TipFunction(fun.args.map(TipType.ast2typevar(_)), fun.stmts.ret.value))
```

A function declaration should add a contraint of function type which is decided by its arguments and return value.
```
X(X1, ..., Xn){ ... return E; }: [X] = ([X1], ..., [Xn]) -> [E]
```
Therefore, the implementation uses the `TipFunction` type constructor with the `TipType` of the node's the arguments and return value to construct the function type.

#### (b)

#### (c)

Given the AST of the program `poly(p) { return *p; }`, the *TypeAnalysis* starts from the program node. When reaching the node of function `poly`, it adds the constraint `[poly] = ([p]) -> [*p]`. Since `[poly]` is a type variable and `([p]) -> [*p]` is a proper type, the *UnionFindSolver* unifies them by settng `([p]) -> [*p]` as the parent of `[poly]` in the directed graph. Next when *TypeAnalysis* visits the value expression of the return statement, it adds another constraint `[p] = &[*p]`. Again, since `[p]` is a type variable and `&[*p]` is a proper type, the *UnionFindSolver* unifies them by settng `&[*p]` as the parent of `[p]`.

After building the *UnionFindSolver*, the *TypeAnalysis* revisits the AST to solve the typing. When it operates on the `poly` function node, it needs to close the node's parent `([p]) -> [*p]` which further requires to close the sub-terms `[*p]` and `[p]` in order. For `[*p]`, since it does not exist in the solver, it is closed as `α`. For `[p]`, since its parent to close is `&[*p]` whose only sub-term is also `[*p]`, it is closed as `&α`. The next AST node to operate is the function argument `p`. Through the same process as before, its type `[p]` is closed as `&α`. The last node need to close is the value `*p` in the return statement. Again, same as before, type variable `[*p]` is closed as `α`.

In the end, the program is processed into
```
[poly]: (&α) -> α
[p]: &α
[*p]: α
```

## Part 2

### 1



