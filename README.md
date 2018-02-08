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

- **`ARM`**: The absolute resolvent method. Parameters:
  - `Eval: RESEVAL_ALG`: The algorithm used to evaluate resolvents
  - `Groups: GROUP_ALG`: The algorithm used to deduce the Galois group from resolvents

- **`SinglyRamified`**: The algorithm due to Greve for singly ramified extensions

- **`Builtin`**: Magma's builtin `GaloisGroup` intrinsic

### `GROUP_ALG`

How to deduce the Galois group using resolvents.

- **`All`**: Enumerate all possible Galois groups, then eliminate possibilities until only one remains. Parameters:
  - `Stat: STATISTIC`: The statistic used to distinguish between groups
  - `Choice: SUBGROUP_CHOICE`: How we choose which subgroups to form resolvents from
- **`Maximal`** Work down the graph of possible Galois groups by maximal inclusion. Parameters:
  - `Stat: STATISTIC`: The statistic used to distinguish between groups; the statistic must support inclusion, not just equality.
  - `Choice: SUBGROUP_CHOICE`: How we choose which subgroups to form resolvents from.
  - `Descend:` How to descend through the graph. One of:
    - **`Eager`**: As soon as a top node is known not to be the Galois group, move on to its children.
    - **`Steady`**: When all top nodes are known to not be the Galois group, move on to their children.
    - **`Patient`**: Like Steady, but also require that the remaining top nodes have a common subgroups which could be a Galois group. Experimental!
    - **`Ask`**: Ask the user.
  - `Useful:`: How to decide whether a subgroup is useful. One of:
    - **`Generous`**: Useful if there is a pair of nodes with different statistics.
    - **`Necessary`**: Useful if it satisfies a necessary condition to provide information.
    - **`Sufficient`**: Useful if it satisfies a sufficient condition to provide information.
    - **`All`**: Always useful.
  - `Reprocess: BOOL`: When true (default), on a descent re-use all resolvents computed so far.
  - `Reset: BOOL`: When true (default), on a descent reset the subgroup choice algorithm.
  - `Blacklist: BOOL`: When true, maintain a blacklist to exclude groups from. Not required if Reprocess is true.
  - `Dedupe: BOOL`: When true, nodes in the graph are merged if they are conjugate.
- **`RootsMaximal`**: Work down the graph of possible Galois groups by maximal inclusion, similar to the relative resolvent method, forming resolvents from the subgroups of the current candidate G and testing for roots to rule out the subgroup or change the candidate to that subgroup. Will compute resolvents of degree equal to the index of the Galois group, which is exponential in the degree of the input polynomial. Parameters:
  - `Dedupe: BOOL`: Dedupe groups by conjugacy.

### `RESEVAL_ALG`

How to evaluate resolvents.

- **`Global`**: Produce a global modeal for the local fields involved. Parameters:
  - `GLOBAL_MODEL`: Specifies how to construct the global model.

### `GLOBAL_MODEL`

How to produce a global model, a global number field which completes to a given local field.

- **`Symmetric`**: Use a global approximation of the defining polynomial, with its coefficients minimized if possible. Assume the global Galois group is the full symmetric group. Parameters:
  - `GALOISGROUP`: Use this algorithm to compute the actual Galois group. This doesn't change the global model, but it can cut down the possibilities for the overall Galois group.
- **`Factors`**: Factorize the polynomial and produce a global model for each factor. Corresponds to a direct product at the group level. Parameters:
  - `GLOBAL_MODEL`: How to produce the model for each factor.
- **`RamTower`**: Get the ramification filtration of the extension defined by the polynomial, and produce a global model for each piece. Corresponds to a wreath product at the group level. Parameters:
  - `GLOBAL_MODEL`: How to produce the model for each piece.
- **`RootOfUnity`**: Adjoin a root of unity to make an unramified extension. The local Galois group is cyclic and the global one is abelian and known. Parameters:
  - `Minimize: BOOL`: By default, we adjoin a `(q^d-1)`th root of unity. When true, minimize the degree of the extension by choosing a suitable divisor of this.
- **`Select`**: Select between several global models. Parameters:
  - `EXPRESSION`: An expression evaluating to true or false in the following variables: `p`, `irr`, `deg`, `unram`, `tame`, `ram`, `wild`, `totram`, `totwild`.
  - `GLOBAL_MODEL`: When the expression is true, use this model.
  - `...`: More expression-model pairs to select between.
  - `GLOBAL_MODEL`: Use this model if no other is used.

### `Statistic`

A function which can be applied to polynomials and groups, with the property that the statistic of a polynomial equals the statistic of its Galois group.

- **`HasRoot`**: True or false depending on whether the resolvent has a root (i.e. the group has a fixed point).
- **`NumRoots`**: The number of roots of the resolvent (i.e. the number of fixed points in the group).
- **`FactorDegrees`**: The multiset of degrees of irreducible factors (i.e. the sizes of the orbits). Equivalent to `Factors[Degree]` but more efficient because it doesn't need to compute the orbit images.
- **`FactorDegrees2`**: Like `FactorDegrees` but also looks at the factors of the factors over the fields defined by the factors.
- **`Factors`**: A multiset of statistics corresponding to the irreducible factors. Parameters:
  - `STATISTIC`: The statistic to use on each factor.
- **`Degree`**: The degree of the polynomial or group.
- **`AutGroup`**: The automorphism group assuming the polynomial is irreducible (i.e. `N_G(S)/S` where `S=Stab_G(1)` up to `S_d`-conjugacy, where `d` is the order of the group, assuming the group is transitive).
- **`Tup`**: A tuple of statistics. Parameters:
  - `STATISTIC`: A statistic.
  - `...`: More statistics.

### `SUBGROUP_CHOICE`

How to choose the subgroup.

- `SUBGROUP_TRANCHE`: Consider each group in each tranche in turn.
- **``**: Consider each group in each tranche in turn, ordered by some priority. Parameters:
  - `SUBGROUP_TRANCHE`: The tranche.
  - `SUBGROUP_PRIORITY`: The priority.

### `SUBGROUP_TRANCHE`

How to select sets of subgroups.

- **`All`**: All subgroups.
- **`Index`**: All subgroups by index. Parameters:
  - `If: EXPRESSION`: Only use the groups where this expression in `idx` evaluates to true.
  - `Sort: EXPRESSION`: Sort the groups by this expression in `idx`.
- **`OrbitIndex`**: All subgroups by orbit-index and index. Orbit-index is the index of the stabilizer of the orbits of the group. Parameters:
  - `If: EXPRESSION`: Only use the groups where this expression in `idx`, `oidx` (orbit index) and `ridx` (remaining index, `idx/oidx`) evaluates to true.
  - `Sort: EXPRESSION`: Sort the groups by this expression in `idx`, `oidx` and `ridx`.

### `SUBGROUP_PRIORITY`

A priority to order a set of groups.

- **`Null`**: Does nothing.
- **`Random`**: Randomizes the order.
- **`Reverse`**: The reverse of its parameter. Parameters:
  - `SUBGROUP_PRIORITY`
- `EXPRESSION`: Sort by this expression in the variables `Index`, `OrbitIndex`, `Diversity`, `Information`.

### `EXPRESSION`

An expression with free variables.

- `<free variable>`: A free variable. The possibilities depend on context.
- `<integer>`: A constant.
- `eq`,`ne`,`le`,`lt`,`ge`,`gt`: Comparisons. Parameters:
  - `EXPRESSION`
  - `EXPRESSION`
- `and`,`or`: True if all or any of the parameters are true. Parameters:
  - `EXPRESSION`
  - `...`
- `-`: Negation. Parameters:
  - `EXPRESSION`
- `-`: Subtraction. Parameters:
  - `EXPRESSION`
  - `EXPRESSION`
- `+`: Sum. Parameters:
  - `EXPRESSION`
  - `...`
- `*`: Product: Parameters:
  - `EXPRESSION`
  - `...`

### `BOOL`

Represents true or false.

- **`True`**
- **`False`**

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
