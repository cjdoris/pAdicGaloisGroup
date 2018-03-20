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
* Optional/recommended: Download and attach the [ExactpAdics](https://cjdoris.github.io/ExactpAdics) package and call `PGG_UseExactpAdicsFactorization()` and `PGG_UseExactpAdicsRoots()`. This makes our package use superior "OM" algorithms for factorization and root finding, instead of the ones builtin to Magma. For polynomials of about degree 32 or more, this can be a significant improvement in both speed and p-adic precision. Note that this does **not** actually use the exact p-adic functionality from the ExactpAdics package (yet).

## Example

The following confirms the Galois group in the 12th line of [this table](http://hobbes.la.asu.edu/LocalFields/basic-table.cgi?prime=2&degree=8) is S_4.

```
> // the following line only needs to be done once per session
> AttachSpec("/path/to/package/spec");
> // optional/recommended if you have the ExactpAdics package (see the Getting started section)
> PGG_UseExactpAdicsFactorization();
> PGG_UseExactpAdicsRoots();
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
- `Maximal [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, DescendWhen, Descend, Useful, Reprocess:BOOL, Reset:BOOL, Blacklist:BOOL, Dedupe:BOOL]`: Work down the graph of possible Galois groups by maximal inclusion.
  - `Stat`: The statistic used to distinguish between possible Galois groups.
  - `Choice`: Determines how to choose which subgroups to form resolvents from.
  - `DescendWhen` When to descend through the graph. One of:
    - `Sufficient`: Don't descend if there are two groups in the pool which might be equal to the Galois group, or if there is one in the pool which might be equal and it has a child which might contain the Galois group. In either of these cases, we can certainly deduce some information (via the `Sufficient` measure of usefulness), so we only descend when we don't know if we can deduce anything.
    - `Always`: Descend as soon as possible.
    - `AllUnequal`: Descend when all nodes in the pool are known not to be the Galois group.
    - `NoSubgroup`: Descend when the subgroup choice algorithm has no subgroups remaining. Note that useful subgroups are marked as "special" in the subgroup choice algorithm, so the subgroup choice algorithm can dynamically change depending on which subgroups are useful.
    - `AllUnequalAndNoSubgroup`: Descend when `AllUnequal` and `NoSubgroup` would both descend. This marks useful subgroups as "special" only when `AllUnequal` would descend.
    - `Necessary`: Like `AllUnequal`, but also require that the remaining top nodes have a common subgroups which could be a Galois group. Experimental!
    - `Ask`: Ask the user.
  - `Descend`: How to descend. One of:
    - `All`: Replace every node in the pool which is not the Galois group by its children.
    - `OneNode`: Replace a single node by its children.
    - `OneChild`: Add a single child to the pool, and remove the parent when all its possible children have been pooled.
  - `Useful:`: How to decide whether a subgroup is useful. One of:
    - `Sufficient`: Useful if there are two groups in the pool which might be equal to the Galois group and which have differing statistics, or if there is one in the pool which might be the Galois group, and a child, such that the statistic of the child is strictly less than the pool group. This guarantees some information is deduced.
    - `Necessary`: Like sufficient, but the statistic of pool group should not be equal to or less than the statistic of the child.
    - `Generous`: Useful if there is a pair of nodes with different statistics.
    - `All`: Always useful.
  - `Reprocess`: When true (default), on a descent re-use all resolvents computed so far.
  - `Reset`: When true (default), on a descent reset the subgroup choice algorithm.
  - `Dedupe`: When true (default), nodes in the graph are merged if they are conjugate.
- `RootsMaximal [Dedupe:BOOL]`: Work down the graph of possible Galois groups by maximal inclusion, similar to the relative resolvent method, forming resolvents from the subgroups of the current candidate G and testing for roots to rule out the subgroup or change the candidate to that subgroup. Will compute resolvents of degree equal to the index of the Galois group, which is exponential in the degree of the input polynomial.
  - `Dedupe`: Dedupe groups by conjugacy.
- `Maximal2 [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, Reset:BOOL, Dedupe:BOOL]`: Like `RootsMaximal` but where the statistic `Roots` is parameterised. We maintain a "pool" of groups such that the Galois group is contained in at least one of them. On each iteration, we find a resolvent such that we can either eliminate all subgroups of some group in the pool, or we can replace a pool element by some of its maximal subgroups. This is very similar to `Maximal` except that instead of merely ruling out groups which cannot contain the Galois group, we identify groups which certainly do contain the Galois group, which is more powerful. `Choice` determines how to choose which subgroups to form resolvents from.
  - `Stat`: The statistic used to distinguish between possible Galois groups.
  - `Choice`: Determines how to choose which subgroups to form resolvents from.
  - `Reset`: When true (default), reset the subgroup choice algorithm each time something gets added to the pool.
  - `Dedupe`: When true (default), groups are deduped by conjugacy.
- `[GROUP_ALG, ...]`: Try each of the algorithms in turn: when the first one runs out of resolvents to try (e.g. because its subgroup choice algorithm is limited) then move on to the second, and so on. This will re-use as much information as possible from one run to the next, so for example `[All[NumRoots,...],All[FactorDegrees,...]]` will only enumerate all possible Galois groups once, and remember which ones were eliminated.
- `ForEach [Vars, [Val1,Val2,...], GROUP_ALG]`: Like the previous, but with a more compact notation. For each of `Val1`, `Val2`, etc, its values are unpacked into variables with names coming from `Vars` and substituted into the `GROUP_ALG`. For example `ForEach[STAT,[NumRoots,FactorDegrees],All[STAT,...]]` is equivalent to the previous example. The `Vars` can be more complex, such as `ForEach[[X,xs],[[A,[a1,a2]],[B,[b]]],ForEach[x,xs,...]]` will have `(x,y)` successively `(A,a1)`, `(A,a2)`, `(B,b)`.

### `RESEVAL_ALG`

How to evaluate resolvents.

- `Global [GLOBAL_MODEL]`: Produce a global model for the local fields involved.

### `GLOBAL_MODEL`

How to produce a global model, a global number field which completes to a given local field.

- `Symmetric [GALOISGROUP]`: Use a global approximation of the defining polynomial, with its coefficients minimized if possible. Assume the global Galois group is the full symmetric group. The `GALOISGROUP` algorithm is used to compute the actual Galois group; this doesn't change the global model, but it can cut down the possibilities for the overall Galois group.
- `Factors [GLOBAL_MODEL]`: Factorize the polynomial and produce a global model for each factor. Corresponds to a direct product of groups.
- `RamTower [GLOBAL_MODEL]`: Get the ramification filtration of the extension defined by the polynomial, and produce a global model for each piece. Corresponds to a wreath product of groups.
- `RootOfUnity [Minimize:BOOL, Complement:BOOL]`: Adjoin a root of unity to make an unramified extension. The local Galois group is cyclic and the global one is abelian and known. By default, we adjoin a `(q^d-1)`th root of unity; when `Minimize` is true, we minimize the degree of the extension by choosing a suitable divisor of this; when `Complement` is set we take a subfield of this so that the global degree is as small as possible. **Note:** The global model may be of higher degree than the local extension, which in a wreath product can make the overall group size exponentially larger.
- `RootOfUniformizer`: Adjoin a root of a uniformizer to make a totally tamely ramified extension. The local and global Galois groups are known.
- `Select [EXPRESSION, GLOBAL_MODEL] ... [GLOBAL_MODEL]`: Select between several global models. The global model next to the first expression evaluating to true is used, or else the final model is used. The expressions are in the following variables: `p`, `irr`, `deg`, `unram`, `tame`, `ram`, `wild`, `totram`, `totwild`.

### `STATISTIC`

A function which can be applied to polynomials and groups, with the property that the statistic of a polynomial equals the statistic of its Galois group.

- `HasRoot`: True or false depending on whether the resolvent has a root (i.e. the group has a fixed point).
- `NumRoots`: The number of roots of the resolvent (i.e. the number of fixed points in the group).
- `FactorDegrees`: The multiset of degrees of irreducible factors (i.e. the sizes of the orbits). Equivalent to `Factors[Degree]` but more efficient because it doesn't need to compute the orbit images.
- `Factors [STATISTIC]`: A multiset of statistics corresponding to the irreducible factors.
- `Factors2 [Stat2:STATISTIC, Stat1:STATISTIC, Strict:BOOL]`: If there are `n` irreducible factors, this is the `n x n` array where the `(i,j)` entry is the multiset of statistics (by `Stat2`) of factors of the `i`th factor over the field defined by the `j`th factor. Each row and column in the array is also labelled with a statistic (by `Stat1`) for the corresponding factor. The array is defined up to a permutation on factors. When `Strict` is true, then two values are equal iff there is a permutation of the factors making the arrays and labels equal; when false, we just check if the multisets along rows and columns are the same, which is much faster (and in practice usually just as good).
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
- `Index [If:EXPRESSION, Sort:EXPRESSION]`: All subgroups by index. Only uses indices where `If` evaluates true, and sorts by `Sort`, both expressions in `idx` (the index). `If` may additionally be in variables `has_special` (true if there exists a special tranche) and `sidx0` (the index of the first special tranche, or 0 if there is none): some tranches may be dynamically marked as special from outside the tranche, e.g. the `Maximal` `GROUP_ALG` with `DescendWhen:NoSubgroup` marks tranches containing useful groups as special, giving a means to dynamically control how many tranches to use based on what was previously useful.
- `OrbitIndex [If:EXPRESSION, Sort:EXPRESSION]`: All subgroups by orbit-index and index. Orbit-index is the index of the stabilizer of the orbits of the group. Only uses indices where `If` evaluates true, and sorts by `Sort`, both expressions in `idx` (the index), `oidx` (orbit index) and `ridx` (remaining index, `idx/oidx`). `If` may additionally be in variables `has_special` and `sidx0`, with the same meanings as for `Index`.

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
- `eq|ne|le|lt|ge|gt [EXPRESSION, EXPRESSION]`: Equality and comparisons.
- `mle|mlt|mge|mgt`: Multiplicative comparisons, e.g. `mle[x,y]` iff `x` divides `y` (note that everything divides 0, so it is the multiplicative infinity)
- `and|or [EXPRESSION, ...]`: True if all or any of the parameters are true.
- `not [EXPRESSION]`: True if its argument is false.
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
- `2by`: the overgroup is of the form C2 wr C2 wr ... (some algorithm parameters are tailored to this)

Each Galois group test also is given a tag for each component of its algorithm, so `ARM[Global[Symmetric],RootsMaximal]` has tags `ARM`, `Global` etc.

### The tests

Currently, there is one type of test called `GaloisGroup`. An instance of this will compute the Galois group of some polynomial over some field using some algorithm, and check that the results equals some expected answer. There is a built-in list of polynomials and a built-in list of algorithms, and we test every polynomial-algorithm pair which is reasonable (i.e. should be compatible and won't take ages to run).
