# Orc Integration

## About Orc

Orc code can be in one of two forms: in .orc files that is converted by
orcc to C code that calls liborc functions, or C code that calls liborc
to create complex operations at runtime. The former is mostly for
functions with predetermined functionality. The latter is for
functionality that is determined at runtime, where writing .orc
functions for all combinations would be prohibitive. Orc also has a fast
memcpy and memset which are useful independently.

## Fast memcpy()

\*\*\* This part is not integrated yet. \*\*\*

Orc has built-in functions `orc_memcpy()` and `orc_memset()` that work
like `memcpy()` and `memset()`. These are meant for large copies only. A
reasonable cutoff for using `orc_memcpy()` instead of `memcpy()` is if the
number of bytes is generally greater than 100. **DO NOT** use `orc_memcpy()`
if the typical is size is less than 20 bytes, especially if the size is
known at compile time, as these cases are inlined by the compiler.

(Example: sys/ximage/ximagesink.c)

Add $(ORC\_CFLAGS) to libgstximagesink\_la\_CFLAGS and $(ORC\_LIBS) to
libgstximagesink\_la\_LIBADD. Then, in the source file, add:

\#ifdef HAVE\_ORC \#include <orc/orc.h> \#else \#define
orc\_memcpy(a,b,c) memcpy(a,b,c) \#endif

Then switch relevant uses of `memcpy()` to `orc_memcpy()`.

The above example works whether or not Orc is enabled at compile time.

## Normal Usage

The following lines are added near the top of Makefile.am for plugins
that use Orc code in .orc files (this is for the volume plugin):

```
ORC_BASE=volume include $(top_srcdir)/common/orc.mk
```

Also add the generated source file to the plugin build:

```
nodist_libgstvolume_la_SOURCES = $(ORC_SOURCES)
```

And of course, add `$(ORC_CFLAGS)` to `libgstvolume_la_CFLAGS`, and
`$(ORC_LIBS)` to `libgstvolume_la_LIBADD`.

The value assigned to `ORC_BASE` does not need to be related to the name
of the plugin.

## Advanced Usage

The Holy Grail of Orc usage is to programmatically generate Orc code at
runtime, have liborc compile it into binary code at runtime, and then
execute this code. Currently, the best example of this is in
Schroedinger. An example of how this would be used is audioconvert:
given an input format, channel position manipulation, dithering and
quantizing configuration, and output format, a Orc code generator would
create an OrcProgram, add the appropriate instructions to do each step
based on the configuration, and then compile the program. Successfully
compiling the program would return a function pointer that can be called
to perform the operation.

This sort of advanced usage requires structural changes to current
plugins (e.g., audioconvert) and will probably be developed
incrementally. Moreover, if such code is intended to be used without Orc
as strict build/runtime requirement, two codepaths would need to be
developed and tested. For this reason, until GStreamer requires Orc, I
think it's a good idea to restrict such advanced usage to the cog plugin
in -bad, which requires Orc.

## Build Process

The goal of the build process is to make Orc non-essential for most
developers and users. This is not to say you shouldn't have Orc
installed -- without it, you will get slow backup C code, just that
people compiling GStreamer are not forced to switch from Liboil to Orc
immediately.

With Orc installed, the build process will use the Orc Compiler (orcc)
to convert each .orc file into a temporary C source (tmp-orc.c) and a
temporary header file (${name}orc.h if constructed from ${base}.orc).
The C source file is compiled and linked to the plugin, and the header
file is included by other source files in the plugin.

If 'make orc-update' is run in the source directory, the files tmp-orc.c
and ${base}orc.h are copied to ${base}orc-dist.c and ${base}orc-dist.h
respectively. The -dist.\[ch\] files are automatically disted via
orc.mk. The -dist.\[ch\] files should be checked in to git whenever the
.orc source is changed and checked in. Example workflow:

edit .orc file ... make, test, etc. make orc-update git add volume.orc
volumeorc-dist.c volumeorc-dist.h git commit

At 'make dist' time, all of the .orc files are compiled, and then copied
to their -dist.\[ch\] counterparts, and then the -dist.\[ch\] files are
added to the dist directory.

Without Orc installed (or --disable-orc given to configure), the
-dist.\[ch\] files are copied to tmp-orc.c and ${name}orc.h. When
compiled Orc disabled, DISABLE\_ORC is defined in config.h, and the C
backup code is compiled. This backup code is pure C, and does not
include orc headers or require linking against liborc.

The common/orc.mk build method is limited by the inflexibility of
automake. The file tmp-orc.c must be a fixed filename, using ORC\_NAME
to generate the filename does not work because it conflicts with
automake's dependency generation. Building multiple .orc files is not
possible due to this restriction.

## Testing

If you create another .orc file, please add it to tests/orc/Makefile.am.
This causes automatic test code to be generated and run during 'make
check'. Each function in the .orc file is tested by comparing the
results of executing the run-time compiled code and the C backup
function.

## Orc Limitations

### audioconvert

Orc doesn't have a mechanism for generating random numbers, which
prevents its use as-is for dithering. One way around this is to generate
suitable dithering values in one pass, then use those values in a second
Orc-based pass.

Orc doesn't handle 64-bit float, for no good reason.

Irrespective of Orc handling 64-bit float, it would be useful to have a
direct 32-bit float to 16-bit integer conversion.

audioconvert is a good candidate for programmatically generated Orc code.

audioconvert enumerates functions in terms of big-endian vs.
little-endian. Orc's functions are "native" and "swapped".
Programmatically generating code removes the need to worry about this.

Orc doesn't handle 24-bit samples. Fixing this is not a priority (for ds).

### videoscale

Orc doesn't handle horizontal resampling yet. The plan is to add special
sampling opcodes, for nearest, bilinear, and cubic interpolation.

### videotestsrc

Lots of code in videotestsrc needs to be rewritten to be SIMD (and Orc)
friendly, e.g., stuff that uses `oil_splat_u8()`.

A fast low-quality random number generator in Orc would be useful here.

### volume

Many of the comments on audioconvert apply here as well.

There are a bunch of FIXMEs in here that are due to misapplied patches.
