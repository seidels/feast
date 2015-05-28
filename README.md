feast:  Bite-sized additions to BEAST 2
=======================================

This is a small [BEAST 2](http://www.beast2.org) package which
contains some additions to the core functionality.  It is compatible
with BEAST 2.2 and higher.

[![Build Status](https://travis-ci.org/tgvaughan/feast.svg?branch=master)](https://travis-ci.org/tgvaughan/feast)


Installation
------------

To install, download the latest release from
[here](https://github.com/tgvaughan/feast/releases) and extract it
into one of the following locations, depending on your operating
system:

 * GNU/Linux: ~/.beast/feast
 * Mac OS X: /library/Application Support/BEAST/feast
 * Windows: #HOME#\BEAST\feast (#HOME# is your user home directory)

More recent releases are also made available as archives containing
only the necessary jar files and the associated javadocs. These are
for the use of BEAST 2 package developers who want to use some of the
classes but don't want to add a full package dependency.  These
archives have the form `*-jarsOnly.zip` and are also found at
https://github.com/tgvaughan/feast/releases.

Alternatively, you can install directly from within BEAUti by adding
the following URL `http://tgvaughan.github.io/feast/package.xml` to
the list of package repositories and selecting feast from the package
list.


Building from source
--------------------

The default target in the provided [Apache ANT](http://ant.apache.org)
build script can be used to build the BEAST 2 package from scratch.
The package will be left in the `dist/` directory.

AlignmentFromNexus/Fasta
------------------------

This class allows alignments to be loaded at runtime from Nexus or Fasta files
rather than stored in the BEAST 2 XML:

```xml
<!-- Nexus: -->
<alignment spec='feast.fileio.AlignmentFromNexus' fileName="example.nexus"/>

<!-- Fasta: -->
<alignment spec='feast.fileio.AlignmentFromFasta' fileName="example.fasta"/>
```

The Fasta import uses the sequence labels from the file as taxon labels.

Obviously these classes runs contrary to BEAST's philosophy of keeping
everything necessary to run an analysis in the XML file.  However, it
is sometimes convenient to be able to do this.  As a bonus, the
`xmlFileName` attribute can be used to write the appropriate XML
fragment to disk.

NexusWriter
-----------

This class complements `NexusParser`, allowing both trees and alignments to be
written to [Nexus files](http://dx.doi.org/10.1093%2Fsysbio%2F46.4.590).
It contains a single static method
```java
public static void write(Alignment alignment, List<Tree> trees, PrintStream ps)
```
that writes the Nexus-formatted data to the given `PrintStream`.

This could be easily used as the basis for a utility which extracts alignments
from BEAST 2 input files.


Input class wrapper
-------------------

The class `feast.input.In` extends `beast.core.Input` and allows
compact creation of Inputs.

All inputs created using this class are by default optional.  The
following creates a basic optional real-valued input:
```java
public Input<Double> xInput = new In<Double>("x", "Tip text.");
```

Default values are set using the method `setDefault()` and rules are
set using the methods `setRequired()` and `setXOR()`.  These methods
return `this` and can therefore be chained (not so useful) or applied
in the input field initialiser (very useful).

For example, the following creates a required boolean input:
```java
public Input<Boolean> yInput = new In<Boolean>("y", "Tip text.").setRequired();
```
while this creates an optional string with default value "default":
```java
public Input<String> yInput = new In<String>("z", "Tip text.").setDefault("default");
```

An XOR-rule is applied between two inputs in the following way:
```java
public Input<Integer> AInput = new In<Integer>("A", "Tip text.");
public Input<Integer> BInput = new In<Integer>("B", "Tip text.").setXOR(AInput);
```

Also, a simple optional input with no default value can be
initialized with the static method `In.create()` which uses type
inference (yes, even in Java 6) to avoid having the input type repeated:
```java
public Input<MyLongObjectName> anInput = In.create("input", "Tip text.");
```
This is equivalent to the the following which uses Java 7's "diamond operator":
```java
public Input<MyLongObjectName> anInput = In<>("input", "Tip text.");
```
Unfortunately, due to limitations in Java, `setDefault()` etc. cannot
be appended in either of these cases. If you require rules or default
values, you'll need to use the long form.

Finally, `feast.input.In` also contains a couple of static methods for
dealing with regular `beast.core.Input` instances.  These include
`In.setRequired()` and `In.setOptional()`.  The most useful of these
is probably `In.setOptional()`, which can be called in a constructor
to make an input that was required in the superclass optional in the
child:
```java
class Parent extends BEASTObject {
      public Input<Integer> reqIntInput = new In<Integer<(
      	     "reqInt", "Tip text.").setRequired();
}

class Child extends Parent {
      Child() {
      	      In.setOptional(reqIntInput);
      }
}
```


Expression Calculator
---------------------

The following classes, which are found in `feast.expressions`, provide
support for parsing of simple arithmetic expressions.


### ExpCalculator ###

Takes an arithmetic expression and returns the result by acting
as a Loggable or a Function.  Binary operators can be applied to
Functions (including Parameters) of different lengths as in R, with
the result having the maximum of the two lengths, and the index into
the shortest parameter being the result index modulo the length of
that parameter.

Example expressions (I and J are IDs of RealParameters with elements
{1.0, 2.0, 3.0} and {5.0, 10.0}, respectively.)

    Expression                 |  Loggable/Function value
    ------------------------------------------------------
    2.5*I                      | {2.5, 5.0, 7.5}
    I+J                        | {6.0, 12.0, 8.0}
    exp(I[0])                  | {2.718...}
    -log(exp(J))/(1.5+0.5*I[0])| {-2.5, -5.0}
    sqrt(J)                    | {2.236..., 3.162...}
    2^I	                       | {2.0, 4.0, 6.0}  
    sum(I)                     | {6.0}
    [1,2,3]^2                  | {1.0, 4.0, 9.0}
    [I,J]                      | {1.0, 2.0, 3.0, 5.0, 10.0}
    theta(I-2)                 | {0.0, 1.0, 1.0}

Note that since each ExpCalculator is a Function object itself, it can
be used as the input for other ExpCalculators.

Inspired by RPNcalculator by Joseph Heled (BEAST 1, BEAST 2 port by
Denise Kuehnert).  (This uses a parser generated by
[ANTLR](http://www.antlr.org), which is cheating.)


### ExpCalculatorDistribution ###

This is basically identical to ExpCalculator, but extends the abstract
Distribution class, allowing the definition of arbitrary distributions
over R^n at runtime.  Distributions can be specified in terms of their
log or directly as probabilities (/probability densities).

The file `ECdistribTest.xml` in the examples/ folder is a simple
example where a multi-variate normal distribution is constructed.
This example also illustrates the nested expression concept mentioned
above.  The file `ECdistribArraytest.xml` in the same folder samples
the same posterior, but uses array notation to express the target
density.


### ExpCalculatorParametricDistribution ###

This provides a cut-down version of the ExpCalculatorDistribution
functionality that is compatible with beast.math.distributions.Prior.


License
-------

This software is free (as in freedom).  With the exception of the
libraries on which it depends, it is made available under the terms of
the GNU General Public Licence version 3, which is contained in this
directory in the file named COPYING.

The following libraries are bundled with Feast:

* ANTLR (http://www.antlr.org)

Those libraries are distributed under the licences provided in the
LICENCE.* files included in this archive.
