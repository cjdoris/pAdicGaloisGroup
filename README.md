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
  , SUBGROUP_TRANCHE       How we choose sets (tranches) of possible subsets to form resolvents from
  , SUBGROUP_ORDER         How we then sort those subgroups
  , SUBGROUP_SCORE         How we then score the subgroups
  , SUBGROUP_CHOICE        How we then choose a subgroup
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
| "FactorDegrees"       The multiset of degrees of irreducible factors (i.e. the sizes of the orbits)

SUBGROUP_TRANCHE
= "All"                 Get all subgroups. Only practical for low degrees.
| "Index"               All subgroups of each index from 1 up to the degree.
| "MostUseful"          Maximal subgroups of subgroups which were previously useful. (Very experimental!)

SUBGROUP_ORDER
= "None"                No reordering.
| "Random"              Shuffle the groups randomly.
| "Index"               Sort by index, lowest first.

SUBGROUP_SCORE
= "IsUseful"            1 if useful, -1 if not.
| "Diversity"           The number of different statistics which appear.
| "Information"         The entropy in the statistics, assuming the Galois group is uniform among possibilities.

SUBGROUP_CHOICE
= "First"               The first.
| "Best"                The highest scoring.
```
