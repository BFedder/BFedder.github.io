---
layout: post
title:  "The EDRReader is (almost) done"
date:   2022-07-31 12:00:00 +0100
categories: jekyll update
---
The last few weeks saw me working on the implementation of an auxiliary reader
for energy data in the EDR format.

# panedr and pyedr
Last time, I reported on my work on panedr and panedrlite, two packages with the
same functionality but different dependencies. While this solution worked, it was
not ideal. A discussion on the pros and cons of different improvements [can be found here](https://github.com/MDAnalysis/panedr/issues/48)
and is recommended reading for anyone trying similar things with packages and their dependencies.
In the end, [IAlibay](https://github.com/IAlibay) implemented the generally preferred solution
in [PR #50](https://github.com/MDAnalysis/panedr/pull/50) and created the panedr
and pyedr packages. A huge Thank You for all the work!


# Problem Statement: EDRReader
The light-weight pyedr package now exists and is ready to be used as a dependency
in MDAnalysis. Finally, the time had come for work on the new AuxReaders to start
in earnest.

The new EDRReader needs to do a number of things:
* Use pyedr to read EDR files and obtain their contents as a dictionary of NumPy arrays
* Make that data accessible via the Reader itself
* Allow association of the data with trajectories
    * The addition of single terms, multiple terms, and all terms should be possible


The base classes in the auxiliary module already take care of a lot of the work here.
Importantly, I did not have to worry much about the association with the correct trajectory frames
itself, thanks to [fiona-naughton's](https://github.com/fiona-naughton) work which she explained [in her blog](https://fiona-naughton.github.io/blog/2016/06/04/Auxiliary-power-then-full-steam-ahead).
The main work lay in adapting these structures to handle the dictionary data type,
and to allow the specification of multiple terms at once. Work on the former point
mostly was focussed within the [EDRReader](https://github.com/MDAnalysis/mdanalysis/blob/ca39385b472ba7ecf9b2e99cc3ddac1800db7590/package/MDAnalysis/auxiliary/EDR.py) itself,
while the latter point required modifying the `add_auxiliary` method in the
[coordinate-base](https://github.com/MDAnalysis/mdanalysis/blob/develop/package/MDAnalysis/coordinates/base.py) module.

# Implementation Details
This work is found in [PR #3749](https://github.com/MDAnalysis/mdanalysis/pull/3749).

Previously, the assumption was that the files to be read by AuxReaders would contain
time-value pairs. It would be called as follows:
```python
u.trajectory.add_auxiliary(auxname, auxdata, format=None, **kwargs):
```
The data found in `auxdata` (either the file itself or an instance of an appropriate AuxReader)
would be added to appropriate time steps with the attribute name `auxname`. This attribute can be
accessed through
```python
u.trajectory.ts.aux.auxname
u.trajectory.ts.aux[auxname]
```
This does not work for EDR files. The problem is that with a larger number of terms,
adding everything under the same `auxname` is not convenient. Therefore, I modified the method
as follows:
```python
u.trajectory.add_auxiliary(auxname, auxdata, auxterm=None, format=None, **kwargs):
```
Users can now specify which of the energy terms to add to the trajectory under `auxname`
by providing an `auxterm` argument to `add_auxiliary`. I had to make it an optional argument
to preserve the XVGReader's functionality.
A list of possible selections for `auxterm` can be obtained from the EDRReader as follows:
```python
aux = MDAnalysis.auxiliary.EDR.EDRReader("some_edr_file.edr")
aux.terms
```
This method now works as follows:
* When no `auxterm` is provided, the auxname must precisely match one of (or a list of) `aux.terms`
* to add a single term, `auxname` and optionally `auxterm` are provided as strings
* to add many terms, `auxname` and optionally `auxterm` are provided as lists of strings
* to add all terms found in the file, the auxname ``"*"`` can be provided.

Examples:
```python
aux = MDAnalysis.auxiliary.EDR.EDRReader("some_edr_file.edr")
# Add the Temperature term to timesteps as a "temp" attribute
u.trajectory.add_auxiliary("temp", aux, "Temperature")

# no auxterm provided, auxname must match one of aux.terms precisely.
# Adds the data as the "Temperature" attribute.
u.trajectory.add_auxiliary("Temperature", aux)

# Adding multiple terms at once (with and without auxterm)
u.trajectory.add_auxiliary(["bond", "temp"], aux, ["Bond", "Temperature"])
u.trajectory.add_auxiliary(["Bond", "Temperature"], aux)

# Adding all data that's in the file
u.trajectory.add_auxiliary("*", aux)
```

Under the hood, a separate instance of an EDRReader is created for each term
that is added to a trajectory. This is necessary to allow updating all auxiliary data
when iterating over the trajectory or otherwise changing the time step. Before I did this,
when an instance of an EDRReader was passed to `add_auxiliary` multiple times,
only the last energy term to be added would be updated when the time step changes.

The EDRReader now is fully functional and allows the association of EDR data with the
appropriate time steps.

# Testing
I have adapted the tests for the auxiliary module for testing the functionality of
the EDRReader and also added further tests for handling more than one term at a time.
There was a small challenge here: Since EDR is a binary file format, I could not generate
a [basic test file](https://github.com/MDAnalysis/mdanalysis/blob/86efe83d77209ef7c6c555a2370cd07589b64a50/testsuite/MDAnalysisTests/auxiliary/base.py#L36) as used for testing the XVGReader. I therefore had to create a small EDR file from a real simulation,
and adapt the tests appropriately. The most significant changes were that the timestep was off,
and that I only added one of the many energy terms for most of the tests.

# Lessons Learned
I was very glad to properly sink my teeth into the code these past few weeks.
This is my largest contribution yet, and I have learned a lot about working with
other peoples' code and building on existing functionality.


# Future Goals
The next steps in this project will be:
* adding type hinting to the new modules/functions
* Implementing a method to obtain the energy data without associating with a trajectory
* Describe how to slice trajectories based on auxiliary data
* Writing the Documentation


Once this is taken care of, I will then move on to an AuxReader for [NumPy arrays](https://github.com/MDAnalysis/mdanalysis/issues/3750)
