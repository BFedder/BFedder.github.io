---
layout: post
title:  "New AuxReaders! But why?"
date:   2022-06-10 12:00:00 +0100
categories: jekyll update
---

# Introduction

Has this ever happened to you? You run an MD simulation with your favourite MD
engine, and you want to make sure the system actually has equilibrated properly
as part of the analysis. In order to inspect the related energy terms, you have
a few hoops to jump through. Depending on your engine of choice you might have to
use a program to extract terms from a binary file, or you might have plain text
file of some formatting or another to deal with. Either way, you will likely have
to write the terms you are interested in to a new file before you can actually work
with them, and you'll have to alt-tab out of your Jupyter Notebook / IDE / what have you.
This may only be mildly annoying, but because it is such a frequent occurence,
even small inefficiencies add up.

In my [GSoC project](https://summerofcode.withgoogle.com/proposals/details/iYc3bcfl),
I will write readers for such energy files and implement them
as part of [MDAnalysis'](https://www.mdanalysis.org) framework for handling
[auxiliary data.](https://userguide.mdanalysis.org/stable/formats/auxiliary.html)
This framework was developed to allow the association of non-trajectory timeseries data
with the frames of a trajectory. One of the key features of the AuxReaders is their
ability to associate auxiliary data with trajectory timesteps when the trajectory data
and the auxiliary data are written with different frequency. If the auxiliary data is
saved, say, half or twice as often, then the AuxReader intelligently assigns it to the
closest trajectory frame. Currently, the XVG format that GROMACS
uses for certain output, for example timeseries data of reaction coordinates in
umbrella sampling or steered MD simulations, is supported.

The new energy readers I will work on this summer will make use of this
framework. Thus, they will not only make inspecting and evaluating energy terms
more convenient, but will also open the door to new ways of utilising energy
data in analysis.

# Concept
The following bits of pseudocode highlight how a new AuxReader for GROMACS EDR files
would work.

Data would be read from files in the same way that the current XVGReader works.
```python
aux = MDAnalysis.auxiliary.EDR.EDRReader("ener.edr")
```
EDR files contain many energy terms, though, where XVG files usually have one data column.
The EDRReader would therefore need to know which data is saved in the energy file.
This information is stored in an attribute.
```python
aux.terms
# Contains a list of all energy terms found in the file
```
Because many terms are present in the files, the method for adding auxdata
to trajectories has to be changed slighly. In addition to providing a name for the
aux attribute, users also specify which of the energy terms to add.
```python
u = mda.Universe(foo, bar)
u.trajectory.add_auxiliary(auxname="epot", auxterm="Potential", auxdata=aux)
```

It will also be possible to add multiple or even all terms found in the file at
the same time.
```python

u.trajectory.add_auxiliary(auxname=["epot", "temp"],
                           auxterm=["Potential", "Temperature"],
                           auxdata=aux)
u.trajectory.add_auxiliary(auxterm=aux.terms, auxdata=aux)
```

Having associated the data with the trajectory, some further analyses become much
easier. It will be very simple, for example, to select subsets of trajectories
based on energy data. For example, to select only those frames of a trajectory
below a certain potential energy threshold:
```python
selected_frames = np.array(
    [ts.frame for ts in u.trajectory if ts.aux.epot < some_threshold])
```
This array of frames can then be used for trajectory slicing.
```python
subset = u.trajectory[selected_frames]
```

Should a user not be interested in such functionality, the energy readers can
also be used to merely "unpack" selected energy terms for plotting.
```python
epot = aux.unpack("Potential")
bond_terms, angle_terms = aux.unpack(["Bond", "Angle"])
```

Having written a few energy readers, I want to compile the lessons I will have learned
doing to into a clear, easy-to-follow tutorial on how to write new AuxReaders to facilitate
further development of this useful but underutilised system (It was first implemented
6 years ago, and only now will new formats be supported).

# [Strategy](https://github.com/MDAnalysis/mdanalysis/issues/3714)
To start this project off, I will work on the implementation of an
[AuxReader for GROMACS' EDR format](https://github.com/MDAnalysis/mdanalysis/issues/3629) for the
selfish reason that GROMACS is the engine I use in my work the most.
[JBarnoud's](https://github.com/jbarnoud) [panedr](https://github.com/MDAnalysis/panedr)
will be crucial for this work. It is a Python package that can read the binary EDR files and
return their content as Pandas DataFrames. It works perfectly with all recent versions
of GROMACS, but Pandas as a dependency should not be introduced to MDAnalysis. My
first task will therefore be a minor rework of panedr so that it returns the energy data
as a dictionary of arrays instead of a DataFrame. Once that is done, I can start working
on a new AuxReader for EDR data that uses the modified panedr and the AuxReader base class
to make the energy data available within MDAnalysis. This will be accompanied by
rigorous documentation and test writing.

In general, two main groups of tests will be needed for the energy readers. For one,
the actual parsing of the energy files has to be tested to make sure the files are
read properly. Secondly, the association of the correct data with the appropriate
trajectory time steps has to be verified.

After finishing work on the EDRReader, I will move on to different MD engines. The
next priority will be either Amber or NAMD. Both of these write energy data to
plain text files, and I can look to a large number of other parsers in MDAnalysis
for inspiration of how to tackle them. I will originally limit myself to one of the formats,
with the option of coming back to work on the other one if I still have time towards
the end of the summer. The reason for this is that these formats are quite similar,
and I ideally want to implement a number of different things.

Next on my list is an AuxReader not for energy data, but instead a general reader
for NumPy arrays. Such a reader would be very interesting because it would greatly
increase the flexibility of the aux module, allowing users to associate any data
of their choice with their trajectories. As an example, users could link the
results of Analysis objects like RMSD values to the timesteps.

Lastly, it would be great to have a CSV reader, as well. This would add yet more
flexibility on one hand, and on the other hand provide a way for OpenMM's energy
data as generated by [StateDataReporters](http://docs.openmm.org/7.0.0/api-python/generated/simtk.openmm.app.statedatareporter.StateDataReporter.html)
to be read.

And finally, as mentioned above, the experience I will have gained in implementing
these AuxReaders should be very helpful in writing detailed documentation and a guide
on how to write additional AuxReaders, helping to make the aux system find more use.

# Timeline
I will be working part-time on this project for a duration of 14 weeks. At the end
of week 6, so roughly in mid- to late-July, I hope to have finished all work on the
EDRReader. Every reader after this one should take less work, and I should be faster, so
that a reader for Amber or NAMD will hopefully be finished by week 9 in mid-August.
This is the bare minimum of what I want to achieve before I start my work on the
guide and documentation, which I will do as the last task before my final submission
deadline on the 26th of September. Depending on how much time I will have left then, I will hopefully
be able to also include the NumPy Reader, the CSV/OpenMM Reader, and the NAMD or Amber
reader, before I start work on the guide.

I am very much looking forward to properly getting started next week! Stay tuned
for the next post on my progress with the EDRReader in a few weeks.
