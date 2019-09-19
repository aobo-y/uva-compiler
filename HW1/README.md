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
This code snippet will check if a program AST node has `main`. If it does, each argument and the return value of the `main` will be unified with type integer. It is unnecessary to add the type constraint `(int, ..., int) -> int` to the `main` function because the function type constraint will be added when the `main` function node is visited as the children of the program node.

The second case is `AAssignStmt`. Its implementation is shown below.

```scala
case ass: AAssignStmt =>
  unify(ass.left, ass.right)
```

It is quite straightforward that the operands on the two sides of the assign operator (`=`) in an assigning statement must belong to the same type. The constraint of assignment is `X = E: [X] = [E]`. Hence, the code simply unifies them.

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

The implementation is tested through typing several build-in examples of TIP which covers all the constructs and the results are manually verified. Also, two error examples which are not typable are tested to see if the analyzer can fail to unify.

The selected examples and their covered constructs are shown below with the typed outputs.

- fib.tip
  - main() { return E; }
  - input
  - E(E1)
  - X(X1) { ... return E; }
  - X = E
  - while(E) S
  - E1 op E2

```
fib(n): (int) -> int
{
var f1: int,f2: int,i: int;
var temp: int;
f1 = 1;
f2 = 1;
i = n;
while ((i > 1)) {
temp = (f1 + f2);
f1 = f2;
f2 = temp;
i = (i - 1);
}
return f2;
}

main(): () -> int
{
var n: int;
n = input;
return fib(n);
}
```

- andersen.tip
  - main() { return E; }
  - X = E
  - &X
  - if (E)
  - *E
  - X() { return E; }
  - null
  - alloc E

```
f(): () -> int
{
var a: &&&α<null:6:17>,b: &&&α<null:6:17>,d: &&α<null:6:17>,e: &&α<null:6:17>,tmp: &&α<null:6:17>;
a = (&d);
b = (&e);
a = b;
tmp = alloc null;
(*a) = tmp;
return 0;
}

main(): () -> int
{
var a: int,b: int,z: &&int,w: &int,x: &int,y: &int,tmp: &int;
z = (&x);
w = (&a);
a = 42;
if ((a > b)) {
tmp = (&a);
(*z) = tmp;
y = (&b);
} else {
x = (&b);
y = w;
}
return 0;
}
```

- foo.tip
  - main() { return E; }
  - input
  - X(X1, X2) { return E; }
  - E(E1, E2)
  - *E
  - E1 == E2
  - if (E)
  - E1 op E2
  - alloc E
  - main

```
foo(p,x): μα<114>.(&int,α<114>) -> int
{
var f: int,q: &int;
if (((*p) == 0)) {
f = 1;
} else {
q = alloc 0;
(*q) = ((*p) - 1);
f = ((*p) * x(q,x));
}
return f;
}

main(): () -> int
{
var n: int;
n = input;
return foo((&n),foo);
}
```

- verybusy.tip
  - main() { return E; }
  - input
  - X = E
  - E1 op E2
  - while (E)
  - output E

```
main(): () -> int
{
var x: int,a: int,b: int;
x = input;
a = (x - 1);
b = (x - 2);
while ((x > 0)) {
output ((a * b) - x);
x = (x - 1);
}
return (a * b);
}
```

The above four examples cover all the constraints and the results are all accurate. Nothing from records is listed here because the records related constraints are given in the templates, not implemented by me. Two other error examples *err_unify1.tip* and *err_unify2.tip* are tested and the analyzer indeed fails to type them. Therefore, I believe the implementation is correct. Although there can be many more other typable and untypable cases to try, they would be the combinations of these tested constructs.

#### (c)

Given the AST of the program `poly(p) { return *p; }`, the *TypeAnalysis* starts from the program node. When reaching the node of function `poly`, it adds the constraint `[poly] = ([p]) -> [*p]`. Since `[poly]` is a type variable and `([p]) -> [*p]` is a proper type, the *UnionFindSolver* unifies them by settng `([p]) -> [*p]` as the parent of `[poly]` in the directed graph. Next when *TypeAnalysis* visits the value expression of the return statement, it adds another constraint `[p] = &[*p]`. Again, since `[p]` is a type variable and `&[*p]` is a proper type, the *UnionFindSolver* unifies them by settng `&[*p]` as the parent of `[p]`.

After building the *UnionFindSolver*, the *TypeAnalysis* revisits the AST to solve the typing. When it operates on the `poly` function node, it needs to close the node's parent `([p]) -> [*p]` which further requires to close the sub-terms `[*p]` and `[p]` in order. For `[*p]`, since it does not exist in the solver, it is closed as `α`. For `[p]`, since its parent to close is `&[*p]` whose only sub-term is also `[*p]`, it is closed as `&α`. The next AST node to operate is the function argument `p`. Through the same process as before, its type `[p]` is closed as `&α`. The last node needs to close is the value `*p` in the return statement. Again, same as before, type variable `[*p]` is closed as `α`.

In the end, the program is processed into
```
[poly]: (&α) -> α
[p]: &α
[*p]: α
```

## Part 2

### 1

The following are the array related constraints.

```
{E1, ..., En}: [E1] = ... = [En] ^ [{E1, ..., En}] = [E1][]
E1[E2]: [E1] = [E1[E2]][] ^ [E2] = int
E1[E2] = E3: [E1] = [E3][] ^ [E2] = int
```

The first type constraint is for array construction. Since the array type is formed with a specific sub-type, every element in the array construction must belong to that same type. Therefore, their type variables are unified in a chain. Correspondingly, the type of the constructor should be the array of this specific type which can be represented by any of the elements as they are the same. The type of the first element `E1` is chosen to represent it.

The second constraint is for element read. The index `E2` must be an integer. The type of the array `E1` must be an array type of this element read `E1[E2]`.

The third constraint is for element write. Again, the index `E2` is for sure an integer. The type of the array `E1` must be an array type of the type of the value `E3` to which its element `E1[E2]` is going to be assigned. However, this constraint is optional, because it is a combination of the ordinary assignment constraint and above element read constraint. The assignment leads to `[E1[E2]] = [E3]` and the element read has `[E1] = [E1[E2]][]`, so they can produce `[E1] = [E3][]`.

### 2

The types of the variables in the sample program are shown in below:
```
var x: int[], y: int, z: int[][], t: int[];
x = {2,4,8,16,32,64};
y = x[x[3]];
z = {{},x};
t = z[1];
t[2] = y;
```
