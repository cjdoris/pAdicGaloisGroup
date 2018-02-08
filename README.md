# pAdicGaloisGroup

Experimental code for computing [Galois groups](https://en.wikipedia.org/wiki/Galois_group) over [p-adic fields](https://en.wikipedia.org/wiki/P-adic_number). Written in [Magma](http://magma.maths.usyd.edu.au/magma).

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

```
GALOISGROUP
= "ARM"                   The absolute resolvent method
  [ Eval: RESEVAL_ALG         The algorithm used to evaluate resolvents
  , Groups: GROUP_ALG         The algorithm used to deduce the Galois group from resolvents
  ]
| "SinglyRamified"        The algorithm due to Greve for singly ramified extensions
| "Builtin"               Use Magma's builtin GaloisGroup intrinsic

GROUP_ALG
= "All"                   Enumerate all possible Galois groups, then try to eliminate possibilities
  [ Stat: STATISTIC           The statistic used to distinguish between groups
  , Choice: SUBGROUP_CHOICE   How we choose which subgroups to form resolvents from
  ]
| "Maximal"               Work down the graph of possible Galois groups by maximal inclusion
  [ Stat: STATISTIC           The statistic used to distinguish between groups
  , Choice: SUBGROUP_CHOICE   How we choose which subgroups to form resolvents from
  , Descend:                  How to descend through the graph
    = "Eager"                     As soon as a top node is known to not be the Galois group, move
                                    on to its children
    | "Steady"                    When all top nodes are known to not be the Galois group, move
                                    on to their children
    | "Patient"                   Like Steady, but also require that the remaining top nodes have
                                    a common subgroup which could be a Galois group
    | "Ask"                       Asks the user
  , Useful:                   How to decide whether a subgroup is useful
    [ "Generous"                  Useful if there is a pair of nodes with different statistics
    , "Necessary"                 Useful if it satisfies a necessary condition to provide information
    , "Sufficient"                Useful if it satisfies a sufficient condition to provide information
    , "All"                       Always useful
    ]
  , Reprocess: BOOL           When true, on a descent re-use all resolvents computed so far
  , Reset: BOOL               When true, on a descent reset the subgroup choice algorithm
  , Blacklist: BOOL           When true, maintain a blacklist to exclude groups from; not required if
                                Reprocess is true
  , Dedupe: BOOL              When true, nodes in the graph are merged if they are conjugate
  ]
| "RootsMaximal"          Work down the graph of possible Galois groups by maximal inclusion, similar
                            to the relative resolvent method, forming resolvents from the subgroups
                            of the current candidate G and testing for roots to rule out the subgroup
                            or change the candidate to that subgroup. Will compute resolvents of degree
                            equal to the index of the Galois group, which is exponential in the degree
                            of the input polynomial.
  [ Dedupe: BOOL              Dedupe groups by conjugacy in the overgroup
  ]

RESEVAL_ALG
= "Global"                Produce a global model for the local fields involved
  [ GLOBAL_MODEL            Specifies how to construct the global model
  ]

GLOBAL_MODEL
= "Symmetric"             Use a global approximation of the defining polynomial, assume the global group is symmetric
  [ GALOISGROUP               Use this algorithm to compute the actual Galois group
  ]
| "Factors"               Factorize the polynomial then...
  [ GLOBAL_MODEL            ... produce a model from each factor in this manner
  ]
| "RamTower"              Get the ramification filtration of the extension defined by the polynomial, then...
  [ GLOBAL_MODEL            ... produce a model from each sub-extension in this manner
  ]
| "RootsOfUnity"          Adjoin a root of unity to make an unramified extension
  [ Minimize: BOOL          When true, minimize the degree of the extension
  ]
| "Select"                Select between several global models
  [ EXPRESSION              An expression evaluating to true or false in the following variables:
                              "p", "irr", "deg", "unram", "tame", "ram", "wild", "totram", "totwild"
  , GLOBAL_MODEL            When the expression is true, use this model.
  ] ...                     More expression-model pairs to select between.
  [ GLOBAL_MODEL            Use this model if no other is used.
  ]

STATISTIC
= "HasRoot"               True or false depending on whether the resolvent has a root (i.e. the group has a fixed point)
| "NumRoots"              The number of roots of the resolvent (i.e. the number of fixed points in the group)
| "FactorDegrees"         The multiset of degrees of irreducible factors (i.e. the sizes of the orbits); equivalent to
                           but more efficient than Factors[Degree]
| "FactorDegrees2"        Like FactorDegrees, but also looks at the factors of the factors over the fields defined by
                            the factors.
| "Factors"               A multiset of statistics corresponding to the irreducible factors
  [ STATISTIC                 The statistic to use on each factor
  ]
| "Degree"                The degree.
| "AutGroup"              The automorphism group (assuming f is irreducible); i.e. N_G(S)/S where S=Stab_G(1)
                            up to S_d conjugacy, where d is the order of the group.
| "Tup"                   A tuple of statistics.
  [ STATISTIC               The list of statistics to use for each component.
  , ...
  ]

SUBGROUP_CHOICE
= SUBGROUP_TRANCHE        Consider each group in each tranche in turn
| [ SUBGROUP_TRANCHE      Consider each tranche in turn, and consider the groups ordered by...
  , SUBGROUP_PRIORITY       ... this priority.
  ]

SUBGROUP_TRANCHE
= "All"                   Consider all subgroups
| "Index"                 Consider subgroups by index
  [ If: EXPRESSION            A filter on the indices to use, with free variable "idx"
  , Sort: EXPRESSION          Sort indices by this expression in "idx"
  ]
| "OrbitIndex"            Consider subgroups by orbit index and index
  [ If: EXPRESSION            A filter on the indices to use, with free variables "idx", "oidx", "ridx"
  , Sort: EXPRESSION          Sort by this expression in "idx", "oidx", "ridx"
  ]

SUBGROUP_PRIORITY         Takes a sequence of possible subgroups and returns it in priority order
= "Null"                    Does nothing
| "Random"                  Random ordering
| "Reverse"                 Reverse of...
  [ SUBGROUP_PRIORITY         ... this
  ]
| EXPRESSION                Sort by this expression, which may have these free variables:
                              "Index", "OrbitIndex", "Diversity", "Information"

EXPRESSION                An expression with some free variables
= <free variable>           A free variable; the possibilities depend on context
| <integer>                 A constant
| ("eq"|"ne"|"le"|"lt"|"ge"|"gt")   Comparisons
  [ EXPRESSION
  , EXPRESSION
  ]
| ("and"|"or")              True if all/any of the arguments are true
  [ EXPRESSION
  , ...
  ]
| "-"                       Negation
  [ EXPRESSION
  ]
| "-"                       Subtraction
  [ EXPRESSION
  , EXPRESSION
  ]
| "+"                       Sum
  [ EXPRESSION
  , ...
  ]
| "*"                       Product
  [ EXPRESSION
  , ...
  ]

BOOL                      A boolean value
= "True"                    True
| "False"                   False
```

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
