# Draft HDF5 units specification

This is not an official standard, and is not endorsed by the HDF group.

## Goals

Data stored in HDF5 files often refers to measurements of physical quantities,
such as distance or energy. Such measurements are only meaningful if their units
are known, and HDF5 attributes provide an obvious place to store units associated
with data.

Units are often seen as something for humans to understand and keep track of,
but there are libraries for multiple programming languages to handle
quantities with units, such as [Boost.Units](http://boost.org/libs/units)
for C++ and [Pint](https://pypi.org/project/Pint/) for Python.
This specification aims to define a format to record units in HDF5 files,
so they can be passed into such libraries without a manual step.

The proposed format is meant to be inspectable by humans with no special
software tools, but it doesn't store units in a particularly user-friendly
presentable format. Software that presents units in a user interface
will most likely want to process them into a more readable form -
some suggestions are given below.

This way of specifying units is loosely inspired by the format for specifying
units in the *casacore* library for astronomy, but it aims to
reduce the number of alternative ways to specify the same units.
Among other things, this means that derived units such as newtons are
not represented directly, but rather as a combination of other units
like kilograms, metres and seconds.
Likewise, metric prefixes are folded into the scaling system.

I haven't tried to accommodate every imaginable unit.
The focus is on consistently expressing many common units.
However, the fractional scaling system can accurately represent
even units like inches and ounces, now that these are defined
in terms of SI units.

## Attributes

Information about units is stored in three attributes, `units`,
`units_scale_numerator` and `units_scale_denominator`.
These attributes may be attached to any numeric dataset (not to e.g. strings).

`units` is a variable-length UTF-8 string, containing a space-separated list of fields.
Each field consists of a unit symbol and an optional integer power
(one or more decimal digits, optionally preceded by an ASCII hyphen-minus to negate it).
Omitting the power part is equivalent to it being 1.

The valid unit symbols are the seven SI base units, plus two dimensionless units
(radians and steradians) whose meaning can't be constructed from other units:

- `m` (metre)
- `kg` (kilogram)
- `s` (second)
- `A` (ampere)
- `K` (kelvin)
- `mol` (mole)
- `cd` (candela)
- `rad` (radian)
- `sr` (steradian)

Unit symbols are case-sensitive, so `M` is not valid, for instance.
They always use symbols, which are standardised, rather than names, which may vary
in different languages.

Thus measurements in joules would be annotated with units `'kg m2 s-2'`.

A numeric dataset may also have one or both of the attributes `units_scale_numerator`
and `units_scale_denominator`. These must be integers. Different integer types may be
used, but if both attributes are present they should use the same type.
The numbers in the dataset are multiplied by `units_scale_numerator` and divided by
`units_scale_denominator` to correspond to the specified units. If either attribute
is not present, it is taken to be 1.

Thus, measurements in kilometres may be expressed with `units='m'` and
`units_scale_numerator=1000`.
Measurements in millimetres may be similarly expressed with `units_scale_denominator=1000`.

Likewise, customary units are described using the scale system.
For example, measurements in inches (25.4 mm) could be recorded
with `units='m'`, `units_scale_numerator=254` and `units_scale_denominator=10000`.
Using two attributes to make a fraction preserves the precise value, as the decimal
number 0.0254 can't be precisely represented as a binary float.

## Advertising compliance

The `units` attribute is widely used without this specification.
Data following this scheme can specify the attribute
`units_scheme="https://url-to-be-determined#1.0"` to declare itself compliant.

This attribute can be used on a dataset with units, or on a group.
Using it on a group declares that all datasets accessible from
that group comply with this specification, including those under
subgroups and those accessed through soft and external links.
Using it on the root group of a file marks the entire file
compliant.

Marking a group as compliant does not necessarily mean that all
(or even any) datasets have units. It means that no datasets use
the attributes specified here in conflicting ways.

Declaring compliance this way is encouraged but not required.
Software reading data of known provenance may expect compliance
even without the marker.

The final part is a version number. This is version 1.0 of the specification.
(N.B. not final - may still change without increasing version number)
Changes such as adding optional attributes or new unit symbols will increase the minor
part of the version number, so any data valid under version 1.0 would also be valid in 1.1.
More dramatic changes would increase the major part.

## Display recommendations

This scheme doesn't represent most quantities in an idiomatic way for human understanding.
Software which shows units to human users will probably want to interpret the
units and present a more human-readable form. In particular:

- Units scaled by powers of 1000 might be represented using metric prefixes,
  such as *km* or *ns*. Masses need special handling to avoid things like
  'kkg'.
- Other specific units may be recognised by scaling, such as hours or litres.
- Where units have more familiar names than a combination of the base units,
  the software may recognise and translate them, such as presenting *V* (volts)
  instead of 'kg m2 s-3 A-1'.

Software designed for a particular domain may recognise additional units
familiar within that domain, such as angstroms.

## Possible additions

- Temperature in Celsius & Fahrenheit can't be expressed by scaling Kelvin.
  We could add these as special exceptions, or allow specifying an offset from 0.
- If we allow specifying a zero-point for time (an epoch), durations could be used to
  represent timestamps (e.g. nanoseconds since the Unix epoch).
- The relationship between radians and degrees involves scaling by an irrational number
  (a fraction of pi), so it cannot be precisely expressed in the fractional scheme
  defined above. If this is a concern, degrees could be added as a separate unit.
- What about units such as bytes which refer to discrete countable things?
  Are these units at all? Arguably e.g. bytes per metre makes sense, but
  so does dots per inch, and we don't consider the dot a unit.

## Rejected options

- Defining symbols for customary units. As an example of the issues, different
  countries disagree substantially on the size of a 'pint'. Many widely used
  customary units, such as inches, pints or ounces, are now formally defined
  in terms of SI units, so they can be precisely represented with the fractional
  scaling mechanism.
- Allowing a slash to invert units, e.g. `'m/s'` instead of `'m s-1'`.
  This would make parsing more complicated.
- Allowing metric prefixes, e.g. `km`. These can be precisely represented by
  fractional scaling. Rejecting them excludes oddities like `kkg`, or
  special handling for kilograms. And if this scheme is ever used for data
  sizes, it avoids the confusion over whether a kilobyte is 1000 or 1024
  bytes.
- I considered an extra attribute for a human-readable unit description,
  but there's significant potential for confusion if the human-readable
  units and the machine-readable units don't match.
- Describing different units for different parts of a single dataset, e.g. if a
  2D array is used for a conceptually 1D list of (longitude, latitude, altitude)
  triples. Working out which units apply to which parts of a multidimensional
  dataset would add significant complexity. It could be added in a future version
  if experience shows that it's a common need.
