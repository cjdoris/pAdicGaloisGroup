# pAdicGaloisGroup

Experimental code for computing [Galois groups](https://en.wikipedia.org/wiki/Galois_group) over [p-adic fields](https://en.wikipedia.org/wiki/P-adic_number). Written in [Magma](http://magma.maths.usyd.edu.au/magma).

We have also compiled some [tables of Galois groups](https://cjdoris.github.io/pAdicGaloisGroupTables).

## Contents
- [Getting started](#getting-started)
- [Example](#example)
- [Verbosity](#verbosity)
- [Algorithm parameter](#algorithm-parameter)
- [Unit testing](#unit-testing)

## Getting started
* [Download from GitHub](https://github.com/cjdoris/pAdicGaloisGroup).
* Attach the `spec` file (see the example below).
* Optional/recommended: Download and attach the [ExactpAdics2](https://cjdoris.github.io/ExactpAdics2) (or [ExactpAdics](https://cjdoris.github.io/ExactpAdics)) package and call `PGG_UseExactpAdicsFactorization()` and `PGG_UseExactpAdicsRoots()`. This makes our package use superior "OM" algorithms for factorization and root finding, instead of the ones builtin to Magma. For polynomials of about degree 32 or more, this can be a significant improvement in both speed and p-adic precision. Note that this does **not** actually use the exact p-adic functionality from the ExactpAdics package (yet).
* Optional/recommended: Download and attach the [ExactpAdics2](https://cjdoris.github.io/ExactpAdics2) (or [ExactpAdics](https://cjdoris.github.io/ExactpAdics)) package and pass a polynomial defined there (e.g. an element of `PolynomialRing(ExactpAdicField(2))`) to `PGG_GaloisGroup`. This causes our algorithm to use the exact p-adic functionality everywhere, providing proven results. It is usually quicker for large inputs.

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

## Main intrinsics

```
GaloisGroup(f :: PGGPol)
PGG_GaloisGroup(f :: RngUPolElt)
PGG_GaloisGroup(f :: RngUPolElt_FldPadExact)
  -> GrpPerm
```

The Galois group of `f` which must be a univariate polynomial defined over a p-adic field.

The group is a permutation group on the roots of `f`, and therefore has the same degree as `f`. The group is defined up to conjugacy in the corresponding symmetric group. If using the absolute resolvent method with global models, it may be defined up to conjugacy in a smaller group depending on the global model used.

If using the `ExactpAdics` package, the polynomial can be over an exact p-adic field.

**Parameters.**
- `Alg`: The algorithm to use (see ["Algorithm parameter"](#algorithm-parameter)).
- `Time`: When true, prints out detailed timings of each part of the algorithm.


## Verbosity

Call `SetVerbose("PGG_GaloisGroup", 1);` to print out information as the algorithm proceeds, including some timings.

## Algorithm parameter

Our implementation is modular, meaning that different algorithms are used in various places. An algorithm is specified by a string of the form `NAME[ARG1,ARG2,...]` where the part in brackets is optional.

The arguments have an order, but in general they have defaults and can be skipped over, so `ARM[All,Global]`, `ARM[All]`, `ARM[Global]` and `ARM` might all be interpreted the same, assuming the arguments to `ARM` have defaults `All` and `Global`.

Arguments can be given by their name instead of by their order, so `ARM[Eval:Global,All]` is interpreted the same as `ARM[All,Global]`.

In fact `:` is syntactic sugar allowing us to write the last parameter outside of brackets. So `a[b,c]:d` is equivalent to `a[b,c,d]`, which is useful for writing highly nested parameters such as `Global:Factors:RamTower:Symmetric:SinglyRamified`.

Here we notate the current options for the algorithms. The `Alg` parameter to `PGG_GaloisGroup` must be a `GALOISGROUP` algorithm.

### `GALOISGROUP`

How to compute a Galois group.

- `ARM [Eval:RESEVAL_ALG, Groups:GROUP_ALG, UpTo:UPTO]`: The absolute resolvent method. Uses `Eval` to evaluate resolvents and `Groups` to deduce the Galois group up to `UpTo`.
- `Tame`: For polynomials whose splitting field is tamely ramified (or unramified).
- `SinglyRamified`: The algorithm due to Greve for singly ramified extensions.
- `Builtin`: Magma's builtin `GaloisGroup` intrinsic. This is currently the "naive" algorithm which computes the splitting field and automorphisms explicitly, so is only suitable for small cases.
- `[GALOISGROUP, ...]`: Tries each algorithm in turn. Useful to rule out special cases, e.g. `[Tame,SinglyRamified]:ARM[...]`.

### `GROUP_ALG`

How to deduce the Galois group using resolvents. Use `Maximal2` in general or `All` when it is quick to enumerate all possible Galois groups.

- `All [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, Dedupe:SUBGROUP_DEDUPE]`: Enumerate all possible Galois groups, then eliminate possibilities until only one remains. `Stat` is the statistic used to distinguish between possible Galois groups. `Choice` determines how to choose which subgroups to form resolvents from. `Dedupe` determines how to dedupe any groups considered up to conjugacy.
- `Maximal [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, DescendWhen, Descend, Useful, Reprocess:BOOL, Reset:BOOL, Blacklist:BOOL, Dedupe:SUBGROUP_DEDUPE]`: Work down the graph of possible Galois groups by maximal inclusion.
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
  - `Dedupe`: How to dedupe any groups considered up to conjugacy.
- `RootsMaximal [Dedupe:SUBGROUP_DEDUPE]`: Work down the graph of possible Galois groups by maximal inclusion, similar to the relative resolvent method, forming resolvents from the subgroups of the current candidate G and testing for roots to rule out the subgroup or change the candidate to that subgroup. Will compute resolvents of degree equal to the index of the Galois group, which is exponential in the degree of the input polynomial.
  - `Dedupe`: How to dedupe any groups considered up to conjugacy.
- `Maximal2 [Stat:STATISTIC, Choice:SUBGROUP_CHOICE, Reset:BOOL, Dedupe:SUBGROUP_DEDUPE]`: Like `RootsMaximal` but where the statistic `Roots` is parameterised. We maintain a "pool" of groups such that the Galois group is contained in at least one of them. On each iteration, we find a resolvent such that we can either eliminate all subgroups of some group in the pool, or we can replace a pool element by some of its maximal subgroups. This is very similar to `Maximal` except that instead of merely ruling out groups which cannot contain the Galois group, we identify groups which certainly do contain the Galois group, which is more powerful. `Choice` determines how to choose which subgroups to form resolvents from.
  - `Stat`: The statistic used to distinguish between possible Galois groups.
  - `Choice`: Determines how to choose which subgroups to form resolvents from.
  - `Reset`: When true (default), reset the subgroup choice algorithm each time something gets added to the pool.
  - `Dedupe`: How to dedupe any groups considered up to conjugacy.
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
- `D4Tower [GLOBAL_MODEL]`: Given an irreducible polynomial of degree 4 defining an extension with Galois group `D4`, splits the extension into a tower of two quadratics and makes a `S2` extension for each one.
- `RootOfUnity [Minimize:BOOL, Complement:BOOL]`: Adjoin a root of unity to make an unramified extension. The local Galois group is cyclic and the global one is abelian and known. By default, we adjoin a `(q^d-1)`th root of unity; when `Minimize` is true, we minimize the degree of the extension by choosing a suitable divisor of this; when `Complement` is set we take a subfield of this so that the global degree is as small as possible. **Note:** The global model may be of higher degree than the local extension, which in a wreath product can make the overall group size exponentially larger.
- `RootOfUniformizer`: Adjoin a root of a uniformizer to make a totally tamely ramified extension. The local and global Galois groups are known.
- `Select [EXPRESSION, GLOBAL_MODEL] ... [GLOBAL_MODEL]`: Select between several global models. The global model next to the first expression evaluating to true is used, or else the final model is used. The expressions are in the following variables: `p` (the prime), `irr` (true if irreducible), `deg` (degree), `unram` (true if defines an unramified extension), `tame` (true if defines a tamely ramified extension), `ram` (true if defines a ramified extension), `wild` (true if defines a wildly ramified extension), `totram` (true if defines a totally ramified extension), `totwild` (true if defines a totally wildly ramified extension).

### `STATISTIC`

A function which can be applied to polynomials and groups, with the property that the statistic of a polynomial equals the statistic of its Galois group. The most useful is usually `FactorDegrees`.

- `HasRoot`: True or false depending on whether the resolvent has a root (i.e. the group has a fixed point).
- `NumRoots`: The number of roots of the resolvent (i.e. the number of fixed points in the group).
- `FactorDegrees`: The multiset of degrees of irreducible factors (i.e. the sizes of the orbits). Equivalent to `Factors[Degree]` but more efficient because it doesn't need to compute the orbit images.
- `FactorDegreesSeq`: A generalization of `FactorDegrees` to a tuple of resolvents (e.g. as returned by the `Tuples` tranche algorithm).
- `Factors [STATISTIC]`: A multiset of statistics corresponding to the irreducible factors.
- `Factors2 [Stat2:STATISTIC, Stat1:STATISTIC, Strict:BOOL]`: If there are `n` irreducible factors, this is the `n x n` array where the `(i,j)` entry is the multiset of statistics (by `Stat2`) of factors of the `i`th factor over the field defined by the `j`th factor. Each row and column in the array is also labelled with a statistic (by `Stat1`) for the corresponding factor. The array is defined up to a permutation on factors. When `Strict` is true, then two values are equal iff there is a permutation of the factors making the arrays and labels equal; when false, we just check if the multisets along rows and columns are the same, which is much faster (and in practice usually just as good).
- `Degree`: The degree of the polynomial or group.
- `AutGroup`: The automorphism group assuming the polynomial is irreducible (i.e. `N_G(S)/S` where `S=Stab_G(1)` up to `S_d`-conjugacy, where `d` is the order of the group, assuming the group is transitive).
- `NumAuts`: The number of automorphisms, i.e. the order of the automorphism group.
- `Tup [STATISTIC, ...]`: A tuple of statistics.
- `Stab [STATISTIC]`: The statistic applied to a point stabilizer of a transitive group (equivalently, the polynomial over the field it defines).
- `GaloisGroup [DegLe:INTEGER, Alg:GALOISGROUP]`: The Galois group, computed using `Alg`, if the degree is at most `DegLe`, and otherwise the trivial group.
- `SubfieldDegrees`: The multiset of degrees of subfields of the field defined by the irreducible polynomial.
- `Order`: The order of the Galois group. Cannot be used on polynomials, only groups.

### `SUBGROUP_CHOICE`

How to choose the subgroup.

- `SUBGROUP_TRANCHE`: Consider each group in each tranche in turn and select the first useful one.
- `[SUBGROUP_TRANCHE, SUBGROUP_PRIORITY]`: As above, but the tranches are reordered according to some priority.
- `Prioritize[SUBGROUP_PRIORITY, SUBGROUP_TRANCHE]`: Equivalent to the previous, but can be used like `Prioritize[PRIORITY]:TRANCHE`.
- `Stream[Limit:INTEGER, SUBGROUP_STREAM]`: Consider up to `Limit` groups from each stream in turn and select the first useful one.

### `SUBGROUP_TRANCHE`

How to select sets ("tranches") of subgroups.

- `All`: All subgroups.
- `Index [If:EXPRESSION, Sort:EXPRESSION, Take:SUBGROUP_TAKE, Dedupe:SUBGROUP_DEDUPE]`: All subgroups by index. Only uses indices where `If` evaluates true, and sorts by `Sort`, both expressions in `idx` (the index). `If` may additionally be in variables `has_special` (true if there exists a special tranche) and `sidx0` (the index of the first special tranche, or 0 if there is none): some tranches may be dynamically marked as special from outside the tranche, e.g. the `Maximal` `GROUP_ALG` with `DescendWhen:NoSubgroup` marks tranches containing useful groups as special, giving a means to dynamically control how many tranches to use based on what was previously useful. `Take` controls which subgroups are used, and `Dedupe` controls how to dedupe groups by conjugacy.
- `OrbitIndex [If:EXPRESSION, Sort:EXPRESSION, Take:SUBGROUP_TAKE, Dedupe:SUBGROUP_DEDUPE]`: All subgroups by orbit-index and index. Orbit-index is the index of the stabilizer of the orbits of the group. Parameters are as for `Index`, except `If` and `Sort` have variables `idx` (index), `oidx` (orbit index) and `ridx` (remaining index, i.e index divided by orbit index).
- `Shuffle [SUBGROUP_TRANCHE]`: The inner tranche, but in a random order.
- `Truncate [Length:INTEGER, SUBGROUP_TRANCHE]`: The inner tranche, but truncated to at most `Length` items.
- `Sample [SUBGROUP_TRANCHE]`: The inner tranche, but takes a random selection of up to `Length` items. (For `Index` or `OrbitIndex`, consider using the parameter `Take:Random` instead.)
- `Tuples [Length:INTEGER, Random:INTEGER, SUBGROUP_TRANCHE]`: Generates tuples of length `Length` from the inner tranche. By default this generates all such tuples, but when `Random` is given, it generates up to this many tuples at random.

### `SUBGROUP_STREAM`

How to select possibly infinite streams of subgroups. Unlike tranches, the subgroups are not cached and so this is typically used for randomly generated subgroups.

- `Index [If:EXPRESSION, Sort:EXPRESSION, Dedupe:SUBGROUP_DEDUPE]`: Random subgroups of a given index. `If` and `Sort` are expressions in `idx` used to filter and sort the indices used (as with `Index` tranche algorithm). `Dedupe` controls how to dedupe groups by conjugacy (currently ignored, no deduping).

### `SUBGROUP_PRIORITY`

A priority to order a set of groups.

- `Null`: Does nothing.
- `Random`: Randomizes the order.
- `Reverse [SUBGROUP_PRIORITY]`: The reverse of its parameter.
- `EXPRESSION`: Sort by this expression in the variables `Index`, `OrbitIndex`, `Diversity`, `Information`.

### `SUBGROUP_DEDUPE`

Controls how to dedupe subgroups up to conjugacy in some larger group.

- `BOOL`: When `False`, no deduping is performed. `True` is equivalent to the current favoured algorithm, currently `Tree`.
- `Pairwise`: Performs pairwise comparisons using `IsConjugate`. Equivalent to `ClassFunction[0]`.
- `ClassFunction [Num:INTEGER]`: Uses a conjugacy-class function to generate a hash for each group so that only groups with the same hash need to be considered. The `Num` selects which function to use, currently one of:
  - 0: Everything hashes to the same value.
  - 1: Counts the number of elements in each conjugacy class of the overgroup.
  - 2: The `GroupName` of the group.
  - 3: The `TransitiveGroupIdentification` of each orbit of the group.
  - 4: The size of the intersection of the group with each normal subgroup of the overgroup up to some index (currently 2).
- `Tree [Statistic:STATISTIC, Tranche:SUBGROUP_TRANCHE]`: Dynamically produces a decision tree to distinguish between groups, the hash being where in the tree the group lies. Each node of the decision tree compares groups by the statistic of their image under the coset action of a group from the tranche algorithm. The default statistic is `Tup[Order,FactorDegrees]` and the default tranche is `OrbitIndex[If:and[mle[idx,p^3],mle[ridx,p]]]` where `p` is the product of primes dividing the order of the overgroup.

### `SUBGROUP_TAKE`

Controls how some tranches generate their groups.

- `All`: Generates all subgroups.
- `Random[Limit:INTEGER, NewTries:INTEGER, RandomTries:INTEGER]`: At most `Limit` randomly-generated subgroups. It tries `RandomTries` times to randomly generate a group. It tries this `NewTries` times to find one we haven't seen before.

### `UPTO`

Specifies an equivalence relation on possible Galois groups such that it suffices to determine the class of the actual Galois group.

In the context of the absolute resolvent method, we take `OvergroupEmbedding[CheckInjective:True]` as the default.

- `OvergroupEmbedding [CheckInjective:BOOL]`: In the absolute resolvent method, with overgroup embedding `e : W -> Wtil`, groups `G1` and `G2` are equivalent if `e(G1)` and `e(G2)` are `Wtil`-conjugate. This is the best we can do, since we then compose this with a coset action `q : Wtil -> A`, which is a class function. If `CheckInjective` is true, then `e` must be injective and therefore this is a finer equivalence than `Symmetric`.
- `Symmetric`: Groups `G1` and `G2` of degree `d` are equivalent if they are `S_d`-conjugate.
- `Full`: In ARM with embedding `e : W -> Wtil`, subgroups `G1` and `G2` of `W` are equivalent if they are `W`-conjugate.

### `EXPRESSION`

An expression with free variables.

- `<free variable>`: A free variable. The possibilities depend on context.
- `<integer>`: A constant.
- `p`: The prime.
- `eq|ne|le|lt|ge|gt [EXPRESSION, EXPRESSION]`: Equality and comparisons.
- `mle|mlt|mge|mgt`: Multiplicative comparisons, e.g. `mle[x,y]` iff `x` divides `y` (note that everything divides 0, so it is the multiplicative infinity)
- `val[p:EXPRESSION,x:EXPRESSION]`, `val[x:EXPRESSION]`: The `p`-adic valuation of `x`.
- `and|or [EXPRESSION, ...]`: True if all or any of the parameters are true.
- `not [EXPRESSION]`: True if its argument is false.
- `- [EXPRESSION]`: Negation.
- `- [EXPRESSION, EXPRESSION]`: Subtraction.
- `+ [EXPRESSION, ...]`: Sum.
- `* [EXPRESSION, ...]`: Product.
- `^ [EXPRESSION, EXPRESSION]`: Power.

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
