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

Our implementation is modular, meaning that different algorithms are used in various places. An algorithm is specified by a string of the form `NAME[ARG1,ARG2,...]` where the part in brackets is optional. The arguments have an order, but in general they have defaults and can be skipped over, so `ARM[All,Global]`, `ARM[All]`, `ARM[Global]` and `ARM` might all be interpreted the same, assuming the arguments to `ARM` have defaults `All` and `Global`.

Here we notate the current options for the algorithms. The `Alg` parameter to `PGG_GaloisGroup` must be a `GALOISGROUP` algorithm.

```
GALOISGROUP
= "ARM"                  The absolute resolvent method
  [ GROUP_ALG              The algorithm used for the group theory parts
  , RESEVAL_ALG            The algorithm used to evaluate resolvents
  , CONJUGACY              Specifies up to what conjugacy the Galois group will be defined
  ]
| "SinglyRamified"       The algorithm due to Greve for singly ramified extensions
| "Builtin"              Use Magma's builtin GaloisGroup intrinsic

GROUP_ALG
= "All"                  Enumerate all possible Galois groups, then try to eliminate possibilities
  [ STATISTIC              The statistic used to distinguish between groups
  , SUBGROUP_CHOICE        How we choose which subgroups to form resolvents from
  ]

RESEVAL_ALG
= "Global"              Produce a global model for the local fields involved

CONJUGACY
= "Symmetric"           The symmetric group
  [ GALOISGROUP           Use this algorithm to compute the actual Galois group
  ]
| "Factors"             Factorize the polynomial then...
  [ CONJUGACY             ... apply this conjugacy to each factor
  ]
| "RamTower"            Get the ramification filtration of the extension defined by the polynomial, then...
  [ CONJUGACY             ... apply this conjugacy to each sub-extension
  ]

STATISTIC
= "HasRoot"             True or false depending on whether the resolvent has a root (i.e. the group has a fixed point)
| "NumRoots"            The number of roots of the resolvent (i.e. the number of fixed points in the group)
| "FactorDegrees"       The multiset of degrees of irreducible factors (i.e. the sizes of the orbits); equivalent to
                          but more efficient than Factors[Degree]
| "FactorDegrees2"      Like FactorDegrees, but also looks at the factors of the factors over the fields defined by
                          the factors.
| "Factors"             A multiset of statistics corresponding to the irreducible factors
  [ STATISTIC              The statistic to use on each factor
  ]
| "Degree"              The degree.
| "AutGroup"            The automorphism group (assuming f is irreducible); i.e. N_G(S)/S where S=Stab_G(1)
                          up to S_d conjugacy, where d is the order of the group.
| "Tup"                 A tuple of statistics.
  [ STATISTIC             The list of statistics to use for each component.
  , ...
  ]

SUBGROUP_CHOICE
= "All"                 Consider all subgroups
  [ SUBGROUP_PRIORITY     How we select among these
  ]
| "Index"               Consider subgroups by index
  [ SUBGROUP_PRIORITY     How we select among these
  , INDEX_PRIORITY        How we select indices
  ]
| "OrbitIndex"          Consider subgroups by orbit index and index
  [ SUBGROUP_PRIORITY     How we select among these
  , OINDEX_PRIORITY       How we select indices
  ]
| "MostUseful"          Maximal subgroups of subgroups which were previously useful. (Very experimental!)
  [ SUBGROUP_PRIORITY     How we select among these
  ]

SUBGROUP_PRIORITY       Takes a sequence of possible subgroups and returns it in priority order
= "Null"                  Does nothing
| "Random"                Random ordering
| "Reverse"               Reverse of...
  [ SUBGROUP_PRIORITY       ... this
  ]
| EXPRESSION              An expression in: "Index", "OrbitIndex", "Diversity", "Information"
| "Filter"                Only keeps items satisfying...
  [ EXPRESSION               ... this expression
  , SUBGROUP_PRIORITY        Then prioritize in this manner
  ]



SUBGROUP_TRANCHE
= "All"                 Get all subgroups. Only practical for low degrees.
| "Index"               All subgroups of each index from 1 up to the degree.
| "MostUseful"          Maximal subgroups of subgroups which were previously useful. (Very experimental!)

SUBGROUP_ORDER
= "None"                No reordering.
| "Random"              Shuffle the groups randomly.
| "Index"               Sort by index, lowest first.
| "OrbitIndex"          Sort by the index of the stablizer of the orbits.
| "Reverse"             The reverse of...
  [ SUBGROUP_ORDER        ... this ordering.
  ]

SUBGROUP_SCORE
= "IsUseful"            1 if useful, -1 if not.
| "Diversity"           The number of different statistics which appear.
| "Information"         The entropy in the statistics, assuming the Galois group is uniform among possibilities.

SUBGROUP_CHOICE
= "First"               The first.
| "Best"                The highest scoring.
```
