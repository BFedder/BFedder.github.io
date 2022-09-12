---
layout: post
title:  "Making Pyedr Optional"
date:   2022-09-12 10:00:00 +0100
categories: jekyll update
---

The EDRReader itself is done and approved now. The final thing left to do at this
point is to make pyedr an optional, not a required dependency for MDAnalysis. This is because
the pyedr package is too large in its current state, with large test files included
in the installation.

# Unit Handling in the EDRReader
My [change to pyedr](https://github.com/MDAnalysis/panedr/pull/56) which made
units available as an output was merged and released, so I was able to include unit handling
in the EDRReader now.

The reader now has an additional `unit_dict` attribute which stores the units of
all terms contained in the `data_dict`. On initialisation, this dictionary is
populated with the output from pyedr, and then checked against the base units of
MDAnalysis. Where the units from the file differ from the MDAnalysis base units,
they are converted automatically. For unit types like pressure or surface tensions,
where no MDAnalysis base units are defined yet, a warning is issued to the user to point
out the potential discrepancies between introduced in the data. It is possible to turn off
automatic unit conversion by setting the `convert_units` kwarg to `False`.

This was the last big change still requested for the EDRReader. With unit handling
implemented, all the desired functionality is now included.

# Finalising the Documentation
In the 2 weeks since my last blog post, I went through the code documentation for
the new features and overhauled and restructured it. It now explains how to use the new
EDRReader much better, and reflects the code properly.

# Making Pyedr Optional
Until such a time when the pyedr package is much smaller than it is now (<1 MB compared
to the current 20 MB), it cannot be a required dependency for MDAnalysis. This might be the case
in the future, for example after [PR #55](https://github.com/MDAnalysis/panedr/pull/55) is merged. For now, to make
pyedr optional (and make sure the tests run or are skipped as appropriate), I applied
what hmacdope has done in [one of his PRs](https://github.com/MDAnalysis/mdanalysis/pull/3765)
for the optional pytng dependency. Basically, the `import pyedr` statement is
now part of a try statement, which assigns a `HAS_PYEDR` the appropriate boolean
depending on whether an `ImportError` is raised or not. Tests that rely on pyedr
are skipped if `HAS_PYEDR` is False, and users who want to use the EDRReader
are advised to install pyedr themselves.

Making pyedr optional also involved changing setup.py, requirements.txt, and the
various CI workflow files.

# Lessons Learned
When changing the code, it is important to check if the documentation is still
accurate. It is much easier to develop a workflow where the documentation is updated
on the go. I learned this the hard way these past two weeks, when a lot of the documentation
I wrote initially had become obsolete and required a rewrite.



# Future Goals
The end of my Summer of Code project is rapidly approaching. My obvious first priority for
now is to get the [EDRReader PR](https://github.com/MDAnalysis/mdanalysis/pull/3749) merged.
Afterwards, it will almost be time to work on my final product for the GSoC evaluation, but if there is time
I will at least start work on the NumPy reader. I will continue work on this project in the future,
so expect to see more AuxReader work to be done even after September.
