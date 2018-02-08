# pAdicGaloisGroup

Experimental code for computing [Galois groups](https://en.wikipedia.org/wiki/Galois_group) over [p-adic fields](https://en.wikipedia.org/wiki/P-adic_number). Written in [Magma](http://magma.maths.usyd.edu.au/magma).

## Contents
- [Getting started](#getting-started)
- [Example](#example)
- [Verbosity](#verbosity)
- [Algorithm parameter](#algorithm-parameter)
- [Unit testing](#unit-testing)

## Getting started
* [Download from GitHub](https://github.com/cjdoris/pAdicGaloisGroup).
* Attach the `spec` file (see the example below).

## Example

The following confirms the Galois group in the 12th line of [this table](http://hobbes.la.asu.edu/LocalFields/basic-table.cgi?prime=2&degree=8) is S_4.

```
> // the following line only needs to be done once per session
> AttachSpec("/path/to/package/spec");
>
> // define a polynomial over Q_2
> K := pAdicField(2,100);
> R<x> := PolynomialRing(K);
> f := x^8 + 20*x^2 + 4;
>
> // compute its Galois group with this package
> // the syntax of the algorithm parameter is explained below
> time G := PGG_GaloisGroup(f : Alg:="ARM[Global[RamTower[Symmetric]],All[FactorDegrees,Index]]");
Time: 0.370
> GroupName(G);
S4
```

## Verbosity

Call `SetVerbose("PGG_GaloisGroup", 1);` to print out information as the algorithm proceeds, including some timings.

## Algorithm parameter

Our implementation is modular, meaning that different algorithms are used in various places. An algorithm is specified by a string of the form `NAME[ARG1,ARG2,...]` where the part in brackets is optional.

The arguments have an order, but in general they have defaults and can be skipped over, so `ARM[All,Global]`, `ARM[All]`, `ARM[Global]` and `ARM` might all be interpreted the same, assuming the arguments to `ARM` have defaults `All` and `Global`.

Arguments can be given by their name instead of by their order, so `ARM[Eval:Global,All]` is interpreted the same as `ARM[All,Global]`.

Here we notate the current options for the algorithms. The `Alg` parameter to `PGG_GaloisGroup` must be a `GALOISGROUP` algorithm.

### `GALOISGROUP`

How to compute a Galois group.

- `ARM [Eval:RESEVAL_ALG, Groups:GROUP_ALG]`: The absolute resolvent method. Uses `Eval` to evaluate resolvents and `Groups` to deduce the Galois group.
- `SinglyRamified`: The algorithm due to Greve for singly ramified extensions.
- `Builtin`: Magma's builtin `GaloisGroup` intrinsic.

### `GROUP_ALG`

How to deduce the Galois group using resolvents.

- `All [Stat:STATISTIC, Choice:SUBGROUP_CHOICE]`: Enumerate all possible Galois groups, then eliminate possibilities until only one remains. `Stat` is the statistic used to distinguish between possible Galois groups. `Choice` determines how to choose which subgroups to form resolvents from.
- `Maximal [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, Descend, Useful, Reprocess:BOOL, Reset:BOOL, Blacklist:BOOL, Dedupe:BOOL]` Work down the graph of possible Galois groups by maximal inclusion. `Stat` is the statistic used to distinguish between possible Galois groups. `Choice` determines how to choose which subgroups to form resolvents from.
  - `Descend` How to descend through the graph. One of:
    - `Eager`: As soon as a top node is known not to be the Galois group, move on to its children.
    - `Steady`: When all top nodes are known to not be the Galois group, move on to their children.
    - `Patient`: Like Steady, but also require that the remaining top nodes have a common subgroups which could be a Galois group. Experimental!
    - `Ask`: Ask the user.
  - `Useful:`: How to decide whether a subgroup is useful. One of:
    - `Generous`: Useful if there is a pair of nodes with different statistics.
    - `Necessary`: Useful if it satisfies a necessary condition to provide information.
    - `Sufficient`: Useful if it satisfies a sufficient condition to provide information.
    - `All`: Always useful.
  - `Reprocess`: When true (default), on a descent re-use all resolvents computed so far.
  - `Reset`: When true (default), on a descent reset the subgroup choice algorithm.
  - `Blacklist`: When true, maintain a blacklist to exclude groups from. Not required if Reprocess is true.
  - `Dedupe`: When true, nodes in the graph are merged if they are conjugate.
- `RootsMaximal [Dedupe:BOOL]`: Work down the graph of possible Galois groups by maximal inclusion, similar to the relative resolvent method, forming resolvents from the subgroups of the current candidate G and testing for roots to rule out the subgroup or change the candidate to that subgroup. Will compute resolvents of degree equal to the index of the Galois group, which is exponential in the degree of the input polynomial.
  - `Dedupe`: Dedupe groups by conjugacy.

### `RESEVAL_ALG`

How to evaluate resolvents.

- `Global [GLOBAL_MODEL]`: Produce a global model for the local fields involved.

### `GLOBAL_MODEL`

How to produce a global model, a global number field which completes to a given local field.

- `Symmetric [GALOISGROUP]`: Use a global approximation of the defining polynomial, with its coefficients minimized if possible. Assume the global Galois group is the full symmetric group. The `GALOISGROUP` algorithm is used to compute the actual Galois group; this doesn't change the global model, but it can cut down the possibilities for the overall Galois group.
- `Factors [GLOBAL_MODEL]`: Factorize the polynomial and produce a global model for each factor. Corresponds to a direct product at the group level.
- `RamTower [GLOBAL_MODEL]`: Get the ramification filtration of the extension defined by the polynomial, and produce a global model for each piece. Corresponds to a wreath product at the group level.
- `RootOfUnity [Minimize:BOOL]`: Adjoin a root of unity to make an unramified extension. The local Galois group is cyclic and the global one is abelian and known. By default, we adjoin a `(q^d-1)`th root of unity; when `Minimize` is true, we minimize the degree of the extension by choosing a suitable divisor of this. **Note:** The global model may be of higher degree than the local extension, which in a wreath product can make the overall group size exponentially larger.
- `RootOfUniformizer`: Adjoin a root of a uniformizer to make a totally tamely ramified extension. The local and global Galois groups are known.
- `Select [EXPRESSION, GLOBAL_MODEL] ... [GLOBAL_MODEL]`: Select between several global models. The global model next to the first expression evaluating to true is used, or else the final model is used. The expressions are in the following variables: `p`, `irr`, `deg`, `unram`, `tame`, `ram`, `wild`, `totram`, `totwild`.

### `Statistic`

A function which can be applied to polynomials and groups, with the property that the statistic of a polynomial equals the statistic of its Galois group.

- `HasRoot`: True or false depending on whether the resolvent has a root (i.e. the group has a fixed point).
- `NumRoots`: The number of roots of the resolvent (i.e. the number of fixed points in the group).
- `FactorDegrees`: The multiset of degrees of irreducible factors (i.e. the sizes of the orbits). Equivalent to `Factors[Degree]` but more efficient because it doesn't need to compute the orbit images.
- `FactorDegrees2`: Like `FactorDegrees` but also looks at the factors of the factors over the fields defined by the factors.
- `Factors [STATISTIC]`: A multiset of statistics corresponding to the irreducible factors.
- `Degree`: The degree of the polynomial or group.
- `AutGroup`: The automorphism group assuming the polynomial is irreducible (i.e. `N_G(S)/S` where `S=Stab_G(1)` up to `S_d`-conjugacy, where `d` is the order of the group, assuming the group is transitive).
- `Tup [STATISTIC, ...]`: A tuple of statistics.

### `SUBGROUP_CHOICE`

How to choose the subgroup.

- `SUBGROUP_TRANCHE`: Consider each group in each tranche in turn.
- `[SUBGROUP_TRANCHE, SUBGROUP_PRIORITY]`: Consider each group in each tranche in turn, ordered by some priority.

### `SUBGROUP_TRANCHE`

How to select sets of subgroups.

- `All`: All subgroups.
- `Index [If:EXPRESSION, Sort:EXPRESSION]`: All subgroups by index. Only uses indices where `If` evaluates true, and sorts by `Sort`, both expressions in `idx` (the index).
- `OrbitIndex [If:EXPRESSION, Sort:EXPRESSION]`: All subgroups by orbit-index and index. Orbit-index is the index of the stabilizer of the orbits of the group. Only uses indices where `If` evaluates true, and sorts by `Sort`, both expressions in `idx` (the index), `oidx` (orbit index) and `ridx` (remaining index, `idx/oidx`).

### `SUBGROUP_PRIORITY`

A priority to order a set of groups.

- `Null`: Does nothing.
- `Random`: Randomizes the order.
- `Reverse [SUBGROUP_PRIORITY]`: The reverse of its parameter.
- `EXPRESSION`: Sort by this expression in the variables `Index`, `OrbitIndex`, `Diversity`, `Information`.

### `EXPRESSION`

An expression with free variables.

- `<free variable>`: A free variable. The possibilities depend on context.
- `<integer>`: A constant.
- `eq|ne|le|lt|ge|gt [EXPRESSION, EXPRESSION]`: Comparisons.
- `and|or [EXPRESSION, ...]`: True if all or any of the parameters are true.
- `- [EXPRESSION]`: Negation.
- `- [EXPRESSION, EXPRESSION]`: Subtraction.
- `+ [EXPRESSION, ...]`: Sum.
- `* [EXPRESSION, ...]`: Product.

### `BOOL`

Represents true or false.

- `True`
- `False`

## Unit testing

The script `unit_tests.mag` can be used to test that most functionality of the package is working. Call it like:

```
magma -b OPTIONS unit_tests.mag
```

### Options

- `Debug:=true/false` (default: `false`) when true, errors during tests cause the debugger to open.
- `Catch:=true/false` (default: `true`) when false, errors during tests are not caught.
- `Quit:=true/false` (default: `true`) when false, keeps Magma open at the end of the script.
- `Start:=N` (default: `1`) starts at the `N`th test.
- `Filter:=STR` STR is a comma-separated list of tags, and we only run the tests which have all of these tags; any tags preceded by a `-` mean that a test must not have this tag.

### Tags

Each test has a set of tags associated to it. Here are their meanings:

- `unram`: unramified
- `tame`: tamely ramified
- `sram`: singly ramified
- `irred`: irreducible
- `degN`: polynomials of degree `N`
- `2by`: the overgroup is of the form C2 wr C2 wr ... (this is because OrbitsOfSubgroup is not implemented in generality yet for wreath products)

Each Galois group test also is given a tag for each component of its algorithm, so `ARM[Global[Symmetric],RootsMaximal]` has tags `ARM`, `Global` etc.

### The tests

Currently, there is one type of test called `GaloisGroup`. An instance of this will compute the Galois group of some polynomial over some field using some algorithm, and check that the results equals some expected answer. There is a built-in list of polynomials and a built-in list of algorithms, and we test every polynomial-algorithm pair which is reasonable (i.e. should be compatible and won't take ages to run).
