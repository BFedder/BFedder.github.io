---
layout: post
title:  "Revenge of the Package"
date:   2022-07-09 12:00:00 +0100
categories: jekyll update
---

In my last post, I wrote about the work I had done on panedr to make it
importable in MDAnalysis, and the challenges that remained to be addressed.
Over the last two weeks, I have made enough progress on the refactoring and
repackaging of panedr that I am now able to start work on the EDRReader in
earnest.


# Refactoring of panedr
In my last blog post, I described how panedr works under the hood by
progressively reading bytes from the binary EDR file and sorting the information
in appropriate Python data structures, ultimately returning a pandas DataFrame.
I am happy to say that my refactoring of this process in [PR #33](https://github.com/MDAnalysis/panedr/pull/33)
is now merged. In addition to a DataFrame, returning the energy data as a dictionary
of NumPy arrays is now supported through `edr_to_dict()`. This function does not
require pandas, making the package optional for this use case.


# Repackaging panedr for dependency management
In order to actually allow the module to be used without pandas installed, the
package structure had to be changed a bit. I worked on this in [PR #42](https://github.com/MDAnalysis/panedr/pull/42).
The boundary conditions to be met were the following:
* `pip install panedr` should work as it does now, installing all dependencies
  for all functions
* Installing the package without pandas needs to be possible
* The code should be in one location and not duplicated

The way I have addressed these conditions is by creating two packages that sit
in parallel in the [panedr repository](https://github.com/MDAnalysis/panedr): panedr
and panedrlite. All the code has moved from panedr to panedrlite, but it's `setup.cfg`
was modified to make pandas an extras_require. Installing panedrlite thus installs
the package and provides most of the functionality out of the box, but does not
install pandas by default. panedr, on the other hand, is an empty metapackage. It
bundles panedrlite and pandas in its dependencies, allowing full functionality on
installation.

However, this is unfortunately not yet the final state for panedr packaging, as my
solution comes with a significant, difficult to fix, problem: installing panedrlite
exposes `import panedrlite`, not `import panedr`. One of the problems this causes
is that it means panedrlite would be installed for users who want to use the functionality
even if they have panedr already installed. There are a [number of possible solutions](https://github.com/MDAnalysis/panedr/issues/48)
to this problem, but they each have drawbacks, and the discussion of which solution is best
is still ongoing.


# Codecov growing pains
When I moved the code from panedr to panedrlite as part of [PR #42](https://github.com/MDAnalysis/panedr/pull/42),
codecov started reporting 0 % coverage. This is weird, because the unit tests
definitely still run and pass (as shown by CI). Initially, we thought the issue might
solve itself on merging the PR, but this was unfortunately not the case, and at the time
of writing, the repository proudly reports 0 % coverage. In the end, IAlibay found a [solution](https://github.com/MDAnalysis/panedr/pull/47)
for the problem, but I have to admit that the reason for why codecov stopped working
and how these changes fix it again are beyond me.


# Work on the EDRReader has started
While some changes are yet to come to panedr, the refactoring is now implemented
and unlikely to change in the near future. As such, I was now finally able to start
work on an [EDRReader implementation](https://github.com/MDAnalysis/mdanalysis/pull/3749).
This is very much a work in progress, but this skeleton now already allows EDR
files to be read in MDAnalysis, which I am happy about.

# Lessons Learned
Now four weeks into my GSoC project, some of the things I learned include:
* there is _always_ more to be learned about packaging
* CI is dark magic and CI experts are wizards
* Having worked on the EDRReader, I have now understood Python's `super()` function better


# Future Goals
The next part of my project will now focus on working on the EDRReader in MDAnalysis.
Because panedr now returns data as a dictionary of numpy arrays, I am also hoping I
might be able to apply some of my work on the EDRReader on a [NumPy AuxReader](https://github.com/MDAnalysis/mdanalysis/issues/3750) as well.
