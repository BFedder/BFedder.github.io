---
layout: post
title:  "Measuring progress"
date:   2022-08-30 12:00:00 +0100
categories: jekyll update
---

Once again, progress on the EDRReader has been made over the last 2 weeks.
It was improved thanks to the incorporation of reviewer feedback, and an important
gap to be filled was identified: Units.

# Monitoring Memory Usage
EDR files can get somewhat large, but the entire file is loaded into memory
for each instance of the EDRReader. This makes monitoring the memory usage useful.
However, there is no straightforward, built-in way in Python to determine the memory footprint of objects,
as [this blog post](https://towardsdatascience.com/the-strange-size-of-python-objects-in-memory-ce87bdfbb97f) neatly illustrates.  
This is because almost everything in Python is an object, and in addition to its value
it has methods and attributes that also take up memory, so there is some overhead to consider.

In the case of the EDRReader, though, this overhead is negligible compared with the size
of the data stored in the EDR files and read into NumPy arrays. In an approximation of
the memory usage, we can therefore rely on the [nbytes attribute](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.nbytes.html)
of NumPy arrays.

In this solution, each `AuxReader` now needs a `_memory_usage` method (and raises `NotImplementedError` if it is missing).
This method returns the total size of the NumPy arrays associated with the `AuxReader`.
When a new instance of an `AuxReader` is associated with a trajectory, the `_memory_usage`
method of all present `AuxReader`s is called and the collective sum compared to
a warning threshold (which defaults to 1 GB). If the memory usage is higher than the threshold,
a warning is issued to the user. This allows for more transparency and better memory control.

The warning threshold can be set via the `memory_limit` kwarg on the creation
of an `AuxReader` instance. This is generally useful, but I needed to implement this
to allow the limit to be lowered sufficiently so tests could trigger the warning without
actually taking up an entire gigabyte of memory. The optional `memory_limit` kwarg
is stored on the level of the `AuxStep`.

# Making the EDRReader Unit-Aware
Part of the discussion on [PR #3749](https://github.com/MDAnalysis/mdanalysis/pull/3749)
centred on the handling of units. This discussion then sparked a discussion how to handle units of auxiliary data
[in general](https://github.com/MDAnalysis/mdanalysis/issues/3792).
It was pointed out that it is desirable for the
data to be converted to MDAnalyis base units on reading. This is currently the last
major change that is requested for the EDRReader before it looks ready to be merged.
Addressing this is not straightforward, though, because the EDRReader is completely unaware of units
at this state.

To change this, I had to go back to the level of [panedr](https://github.com/MDAnalysis/panedr).
(Once again learning the lesson that nothing is ever truly done; I was too optimistic
when I opened PR #3749 and said pyedr's "actual code itself [was] less likely to still change")
This is because in the process of reading the EDR file, panedr/pyedr were at some point
aware of the units of each entry, but did not retain this information.

During the execution of pyedr, the `nms` variable is populated with the names and units found in the
EDR file by the `edr_strings()` function. I have changed the package in [PR #56](https://github.com/MDAnalysis/panedr/pull/56)
to also make the unit information available as an output.

Now, users of panedr and pyedr can call the `get_unit_dictionary` function to obtain
a dictionary of the units of each energy term found in the EDR file as follows:

```python
import pyedr
edr_file = "path/to/some/edrfile.edr"

unit_dict = pyedr.get_unit_dictionary(edr_file)

unit_dict["Temperature"]    # Returns "K"
```

I am hoping that this PR can be merged and the updated package released soon,
so that I can then work on making the units available on the level of the `EDRReader`
as well.

# Lessons Learned
I learned more about how to accommodate `kwargs` that are meant to be passed as
optional arguments, and I learned about the [pint package](https://pint.readthedocs.io/en/stable/)
for unit handling in Python.



# Future Goals
  - Get [PR #56](https://github.com/MDAnalysis/panedr/pull/56) merged
  - Change the EDRReader to store unit information alongside the data themselves
  - Implement automatic conversion to MDAnalysis base units
  - Start work on the auxiliary reader for NumPy arrays
