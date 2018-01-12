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
> time G := PGG_GaloisGroup(f : Alg:="ARM[All[FactorDegrees,Index],Global,RamTower[Symmetric]]");
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
  [ Groups: GROUP_ALG         The algorithm used for the group theory parts
  , Eval: RESEVAL_ALG         The algorithm used to evaluate resolvents
  , Conj: CONJUGACY           Specifies up to what conjugacy the Galois group will be defined
  , UseEasyResolvents: BOOL   When true, start by using cheap resolvents
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
  , Reprocess: BOOL           When true, on a descent re-use all resolvents computed so far
  , Reset: BOOL               When true, on a descent reset the subgroup choice algorithm
  , Useful:                   How to decide whether a subgroup is useful
    [ "Generous"                  Useful if there is a pair of nodes with different statistics
    , "Necessary"                 Useful if it satisfies a necessary condition to provide information
    , "Sufficient"                Useful if it satisfies a sufficient condition to provide information
    ]
  ]
| "RootsMaximal"          Work down the graph of possible Galois groups by maximal inclusion, similar
                            to the relative resolvent method, forming resolvents from the subgroups
                            of the current candidate G and testing for roots to rule out the subgroup
                            or change the candidate to that subgroup. Will compute resolvents of degree
                            equal to the index of the Galois group, which is exponential in the degree
                            of the input polynomial.

RESEVAL_ALG
= "Global"                Produce a global model for the local fields involved

CONJUGACY
= "Symmetric"             The symmetric group
  [ GALOISGROUP               Use this algorithm to compute the actual Galois group
  ]
| "Factors"               Factorize the polynomial then...
  [ CONJUGACY                 ... apply this conjugacy to each factor
  ]
| "RamTower"              Get the ramification filtration of the extension defined by the polynomial, then...
  [ CONJUGACY                 ... apply this conjugacy to each sub-extension
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
= "All"                   Consider all subgroups
  [ SUBGROUP_PRIORITY       How we select among these
  ]
| "Index"                 Consider subgroups by index
  [ SUBGROUP_PRIORITY         How we select among these
  , If: EXPRESSION            A filter on the indices to use, with free variable "idx"
  , Sort: EXPRESSION          Sort indices by this expression in "idx"
  ]
| "OrbitIndex"            Consider subgroups by orbit index and index
  [ SUBGROUP_PRIORITY         How we select among these
  , If: EXPRESSION            A filter on the indices to use, with free variables "idx", "oidx", "ridx"
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

BOOL                      A boolean value
= "True"                    True
| "False"                   False
```