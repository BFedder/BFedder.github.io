---
layout: post
title:  "Overhauling the Auxiliary API"
date:   2022-08-14 12:00:00 +0100
categories: jekyll update
---
My work on the EDRReader has given us some ideas on how the process of attaching
auxiliary data could be changed in general, and that is what I have been working
on this week.

## Adding Auxiliary Data, Take One
As a reminder, this is how adding EDR data worked as of my previous blog post on my work in
[PR #3749](https://github.com/MDAnalysis/mdanalysis/pull/3749):
```python
u.trajectory.add_auxiliary(auxname, auxdata, auxterm=None, format=None, **kwargs):
```

* When no `auxterm` is provided, the auxname must precisely match one of (or a list of) `aux.terms`
* to add a single term, `auxname` and optionally `auxterm` are provided as strings
* to add many terms, `auxname` and optionally `auxterm` are provided as lists of strings
* to add all terms found in the file, the auxname ``"*"`` can be provided.

This worked, but it wasn't ideal. A number of problems with this approach existed,
including but not limited to:
* It was confusing to have `auxname` have different requirements depending on whether
  or not `auxterm` was provided
* The fact that strings and lists could be provided for both of these arguments
  required a number of checks, adding a lot of code
* Using `"*"` to signify 'everything' is not very pythonic. A more natural way
  to add all data found in the file is to simply specify `auxdata`. This means
  `auxname` should be optional and default to `None`.
* `MDAnalysis.coordinates.base.add_auxiliary` was made quite complicated by checking
  for the type of AuxReader in `auxdata` and calling a separate method for attaching
  the data for each Reader. This would have been especially problematic on addition of
  more and more readers.

## Adding Auxiliary Data, Take Two
# AuxReader who?
One thing I did to improve on this is to move where in the code the auxiliary
data actually is associated with the trajectory. Previously, `MDAnalysis.coordinates.base.add_auxiliary`
would add the auxiliary data itself, and I had then changed it so the method would instead
call the appropriate file format specific method.

We have discussed this change, and really, there is no reason why `add_auxiliary()`
should need to know about which format it is dealing with, or even which formats
currently are supported. Instead, the code now makes use of a useful code pattern:

```python
class Parent:

	def upper_method(self, child_obj, ...):
    	child_obj.lower_method(self, ...)

class Child:

	def lower_method(self, parent_obj):
    	parent_obj.add_attribute(...)
```

Applied to MDAnalysis, the `Parent` class is the trajectory of the Universe to which
auxiliary data is added, and the `upper_method` is `add_auxiliary`. The role of the `Child` class
is taken by the AuxReader Base class.

What this pattern allows is the manipulation of the `Parent` object through the `Child`
object, as it passes it`self` to its child through `upper_method`. It is thus available to
the `Child`.

I have now written an `attach_auxiliary()` method of the base AuxReader class, which
receives its parent trajectory as an argument and allows attaching correctly configured
AuxReader instances to this parent. This simplified `add_auxiliary` massively. Also, the
auxiliary module is a more natural home for the code that does the actual association than the coordinate module.

# Resolving the `auxterm`-`auxname`-Confusion
To avoid the confusion that the inconsisten requirements of what `auxname` should be,
the two parameters were replaced by a single `aux_spec` parameter.

`aux_spec` should be a dictionary that maps the desired attribute names to the precise
names of the data in the auxiliary file (formerly `auxterm`). At the same time, this
removed the need to check if one or many terms are to be added, so whether a string or a list
is provided.

The way this works now is as follows:
```python
import MDAnalysis as mda
from MDAnalysisTests.datafiles import AUX_EDR, AUX_EDR_TPR, AUX_EDR_XTC
term_dict = {"temp": "Temperature", "epot": "Potential"}
aux = mda.auxiliary.EDR.EDRReader(AUX_EDR)
u = mda.Universe(AUX_EDR_TPR, AUX_EDR_XTC)
u.trajectory.add_auxiliary(term_dict, aux)
```

There is one exception to this behaviour: For reasons of backwards compatibility,
it is also possible to provide a string as `aux_spec`. In this case, the `attach_auxiliary`
method will create a dictionary mapping this string value to `None`. This maintains the current default
behaviour of the XVGReader, where passing a string as `auxname` causes everything found in the XVG
file to be added under `auxname`. The value of the `aux_spec` key-value-pairs is
set to the AuxReaders `data_selector`, with `None` causing everything to be added.
This pattern will also generalise well to other file formats. The values of the dictionary could, for
example, also be column indices of a CSV file.

# More Pythonic Syntax
Adding all terms by providing `"*"` as an argument is not very pythonic. A better
behaviour would be one were providing no `aux_spec` at all would add everything.
This could be implemented by making `aux_spec` an optional argument that defaults to `None`.
However, there is a problem: `auxdata` isn't optional, so the order of the function parameters has to be reversed:
```python
add_auxiliary(auxname, auxdata)
```
would become
```python
add_auxiliary(auxdata, auxname=None)
```
But because the XVGReader currently expects `auxname` as the first argument and `auxdata` as the second,
this change would break the XVGReader.

I identified four possible solutions for this:
  1. We change the behaviour of the XVGReader
  2. We use unpythonic syntax
  3. We explicitly provide None as an argument to add everything, which is the opposite of what I would expect.
  4. We make both `auxname` and `auxdata` optional and default to `None`, but raise an exception when no `auxdata` is provided.

Currently, the [PR](https://github.com/MDAnalysis/mdanalysis/pull/3749) reflects solution 4 and looks like this:
```python
def add_auxiliary(self,
                  aux_spec: Union[str, Dict[str, str]] = None,
                  auxdata: Union[str, AuxReader] = None,
                  format: str = None,
                  **kwargs) -> None:
```
This implementation seemed to me the best option. The XVGReader still works as before because
the order of the parameters is maintained. The `"*"` no longer is an option, therefore the code is more pythonic.
And not specifying `aux_spec` and therefore passing `None` implicitly works as expected, adding all data.
There is a minor downside: Because `auxdata` is the second parameter of `add_auxiliary` (and cannot be first parameter to maintain XVGReader functionality),
`auxdata=aux` has to be passed as argument, rather than just `aux`, when no `aux_spec` is provided.
To add everything, the method is now used as follows :

```python
u.trajectory.add_auxiliary(auxdata=aux)
```


# Lessons Learned
The code pattern of a Parent class instance passing itself to a Child class instance is super useful!

Saying something is ["(almost) done"](https://bfedder.github.io/jekyll/update/2022/07/31/EDRReader-done.html) before review is unwise




# Future Goals
With these changes, the most critical issues mentioned in the PR reviews are addressed.
Next, I'll address the remaining comments, which should then move the EDRReader closer to being merged.
Thereafter, work on the [NumPy array reader](https://github.com/MDAnalysis/mdanalysis/issues/3750) will start.
