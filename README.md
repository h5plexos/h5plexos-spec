# H5PLEXOS Data Format Specification

_Note: A useful reference for HDF5 file structure concepts is the
[HDF5 Glossary](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary).
This document contains links to glossary entries to explain HDF5 terms when used
for the first time._

This file defines __version 0.6.1__ of the H5PLEXOS data format specification.
This specification outlines a standardized means of storing results from a
PLEXOS energy system simulation in terms of
[objects](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Object)
in an HDF5 file. It does not attempt to describe how those results, once stored
in HDF5, should be presented, queried, retreived, or otherwise made available
in any particular programming language or interactive environment.

## PLEXOS Schema Background

This specification assumes some familiarity with the relational structure
used by PLEXOS to represent energy system components. Most notably:

 - PLEXOS objects represent various elements of the energy system model. An
   object belongs to a particular PLEXOS class.

 - PLEXOS memberships define relationships between two PLEXOS objects. In a
   membership, one object is considered the "parent" object while the other is
   the "child" object.

 - A PLEXOS collection defines a set of memberships between specific
   classes of PLEXOS objects.

 - PLEXOS properties define characteristics of memberships belonging to a
   particular PLEXOS collection.

 - Numerical PLEXOS result outputs are provided for a specific property (and
   thus collection), a specific membership in the collection, specific time
   period, and specific band. (Many properties report data on a single band,
   although some use multiple bands.)   

## Collection Naming and Classification Conventions

The following section explains how H5PLEXOS assigns names to PLEXOS
collections. Before being stored in an H5PLEXOS file, any collection names
provided by PLEXOS should have spaces stripped
out, and all other characters converted to lowercase ASCII symbols.

### Object collections

In H5PLEXOS, collections of memberships containing parent objects of class
"System" (such that the parent object in all the collection's memberships is
the same singleton object "System") are considered to be "object" collections
(the parent object provides no information, so the collection just describes
the set of child objects). These collections are given
H5PLEXOS names identical to their original PLEXOS collection names.

### Relation collections

In H5PLEXOS, collections of memberships containing parent objects of any class
other than "System" are considered to be "relation" collections (memberships
belonging to these collections use both the
parent and child objects to express relational information).

If PLEXOS provides a "complement name" for the collection, H5PLEXOS names the
collection as the complement name, followed by an underscore, followed by the
original PLEXOS collection name.

If PLEXOS does not provide a "complement name" for the collection, H5PLEXOS
names the collection as the parent class' name, followed by an underscore,
followed by the original PLEXOS collection name.

## H5PLEXOS HDF5 file structure specification

The
[root group](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Rootgroup)
of an H5PLEXOS-compatible HDF5 file contains two child
[groups](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Group)
`metadata` and `data`, and also has various
[attributes](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Attribute)
corresponding to the model's supplementary metadata (`Version`,
`Username`, `Computer`, etc) as reported by PLEXOS.

### `/metadata`

The `/metadata` group contains information that can be used to contextualize
the numerical results stored in the sibling `/data` group. It contains three
child groups, `objects`, `relations`, and `times`.

#### `/metadata/objects`

The `/metadata/objects` group contains one
[dataset](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Dataset)
for each _object collection_ in the PLEXOS result data.

Each dataset (`/metadata/objects/{collection}`) is a one-dimensional array of
[compound datatypes](https://portal.hdfgroup.org/display/HDF5/HDF5+Glossary#HDF5Glossary-Compounddatatype)
with two fields each:

  1. `name`: An ASCII string storing the name of the membership's child object
  2. `category`: An ASCII string storing the category of the membership's child object

#### `/metadata/relations`

The `/metadata/relations` group contains one dataset
for each _relation collection_ in the PLEXOS result data.

Each dataset (`/metadata/relations/{collection}`) is a one-dimensional array of
compound datatypes with two fields each:

  1. `parent`: An ASCII string storing the name of the membership's parent object
  2. `child`: An ASCII string storing the name of the membership's child object

#### `/metadata/times`

The `/metadata/times` group contains one dataset
for each result period type (aggregation level) in the PLEXOS result data.
(Example period types are interval, hourly, daily, weekly, etc.)

Each dataset (`/metadata/times/{period}`) is a one-dimensional array of
strings labelling the time periods at that aggregation level. The strings
should be ASCII formatted and 19 characters long, with the timestep provided as
an ISO-8601 date and time (`yyyy-mm-ddTHH:MM:SS`). The value of the label
should be converted from the corresponding value
originally reported by PLEXOS (which may be in a non-IS0-8601 datetime format).

### `/data`

The `/data` group contains the numerical results reported by a PLEXOS
simulation. These results are stored in property-specific datasets and
organized in groups by the corresponding phase (ST, MT, PASA, LT), period
(interval, hour, day, etc), and collection (generators, lines,
reserve_generators, etc).

A `/data/{phase}/{period}/{collection}` group contains datasets for each
property reported for that combination of phase, period, and collection.
Each dataset is a three-dimensional array of floating point numbers and has
two attributes defined on it: `units` and `period_offset`.

The `units` attribute provides a string describing the physical or economic
units in which the dataset's values are stored.

The `period_offset` attribute provides an integer describing the extent to
which the provided data is temporally offset into the
`/metadata/times/{period}` timestamps (this is discussed in more detail below).

The size of the outer dimension of the array (first dimension in C/HDF5
row-major format, last dimension in Julia/MATLAB/Fortran column-major
format) should match the size of the collection (the number of objects or
relations) the property is defined over. Data in this axis should be ordered
according to the elements of the corresponding
`/metadata/objects/{collection}` or `/metadata/relations/{collection}` dataset.
For example, all the values in the third position along the object axis of
`/data/{phase}/{period}/{collection}/{property}` correspond to the object
described by the third position of `/metadata/objects/{collection}`.

The size of the middle dimension of the array (the second in both C/HDF5
and Julia/MATLAB/Fortran formats) should match the number of time periods for
which the property reports data. Note that this may be less than the total
number of periods provided in `/metadata/times/{period}` - in this case the
`period_offset` attribute is used to indicate the (zero-indexed) starting
location in `/metadata/times/{period}`. Data should be chronologically
increasing with respect to this array axis. For example, all the values in the
third position along the time axis of
`/data/{phase}/{period}/{collection}/{property}` correspond to the period
described by the third + `period_offset` position of `/metadata/times/{period}`.

The size of the innermost dimension of the array (the last dimension in C/HDF5
row-major format, and the first dimension in Julia/MATLAB/Fortran column-major
format) should match the number of bands used to report the property
(often just one), with entity data provided in ascending band order. For
example, all the values in the third position along the band axis of
`/data/{phase}/{period}/{collection}/{property}` correspond to the third band
of the results.
