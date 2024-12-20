---
Authors: Bradley Chambers, Scott Lewis
Contact: <mailto:brad.chambers@gmail.com>
Date: 11/02/2017
---

(writing-reader)=

# Writing a reader

PDAL's command-line application can be extended through the development of
reader functions. In this tutorial, we will give a brief example.

## The header

First, we provide a full listing of the reader header.

```{literalinclude} ../../examples/writing-reader/MyReader.hpp
:language: cpp
:linenos: true
```

```{literalinclude} ../../examples/writing-reader/MyReader.hpp
:language: cpp
:linenos: true
:lines: 18-20
```

`m_stream` is used to process the input, while `m_index` is used to track
the index of the records.  `m_scale_z` is specific to MyReader, and will
be described later.

```{literalinclude} ../../examples/writing-reader/MyReader.hpp
:language: cpp
:linenos: true
:lines: 22-26
```

Various other override methods for the stage.  There are a few others that
could be overridden, which will not be discussed in this tutorial.

```{note}
See `./include/pdal/Reader.hpp` of the source tree for more methods
that a reader can override or implement.
```

## The source

Again, we start with a full listing of the reader source.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
```

In your reader implementation, you will use a macro to create the plugin.
This macro registers the plugin with the PDAL PluginManager.  In this case,
we are declaring this as a SHARED stage, meaning that it will be loaded at
runtime instead of being linked
to the main PDAL installation.  The macro is supplied with the class name
of the plugin and a PluginInfo object.  The PluginInfo objection includes
the name of the plugin, a description, and a link to documentation.

When making a shared plugin,
the name of the shared library must correspond with the name of the reader
provided here.  The name of the generated shared object must be

```
libpdal_plugin_reader_<reader name>.<shared library extension>
```

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 8-15
```

This method will process a options for the reader.  In this
example, we are setting the z_scale value to a default of 1.0, indicating
that the Z values we read should remain as-is.  (In our reader, this could
be changed if, for example, the Z values in the file represented mm values,
and we want to represent them as m in the storage model). `addArgs` will
bind values given for the argument to the `m_scale_z` variable of the
stage.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 19-22
```

This method registers the various dimensions the reader will use.  In our case,
we are using the X, Y, and Z built-in dimensions, as well as a custom
dimension MyData.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 24-30
```

This method is called when the Reader is ready for use.  It will only be
called once, regardless of the number of PointViews that are to be
processed.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 32-36
```

This is a helper function, which will convert a string value into the type
specified when it's called.  In our example, it will be used to convert
strings to doubles when reading from the input stream.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 38-52
```

This method is the main processing method for the reader.  It takes a
pointer to a PointView which we will build as we read from the file.  We
initialize some variables as well, and then reset the input stream with
the filename used for the reader.  Note that in other readers, the contents
of this method could be very different depending on the format of the file
being read, but this should serve as a good start for how to build the
PointView object.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 57-62
```

In preparation for reading the file, we prepare to skip some header lines.  In
our case, the header is only a single line.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 64-66
```

Here we begin our main loop.  In our example file, the first line is a header,
and each line thereafter is a single point.  If the file had a different format
the method of looping and reading would have to change as appropriate.  We make
sure we are skipping the header lines here before moving on.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 67-72
```

Here we take the line we read in the for block header, split it, and make sure
that we have the proper number of fields.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 75-84
```

Here we take the values we read and put them into the PointView object.  The
X and Y fields are simply converted from the file and put into the respective
fields.  MyData is done likewise with the custom dimension we defined.  The Z
value is read, and multiplied by the scale_z option (defaulted to 1.0), before
the converted value is put into the field.

When putting the value into the PointView object, we pass in the Dimension
that we are assigning it to, the ID of the point (which is incremented in
each iteration of the loop), and the dimension value.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 86-99
```

Finally, we increment the nextId and make a call into the progress callback
if we have one with our nextId.  After the loop is done, we set the index
and number read, and return that value as the number of points read.
This could differ in cases where we read multiple streams, but that won't
be covered here.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 101-108
```

When the read method is finished, the done method is called for any cleanup.
In this case, we simply make sure the stream is reset.

```{literalinclude} ../../examples/writing-reader/MyReader.cpp
:language: cpp
:linenos: true
:lines: 111-114
```

## Compiling and Usage

The MyReader.cpp code can be compiled.  For this example, we'll use cmake.
Here is the CMakeLists.txt file we will use:

```{literalinclude} ../../examples/writing-reader/CMakeLists.txt
:linenos: true
```

If this file is in the directory containing MyReader.hpp and MyReader.cpp,
simply run `cmake .`, followed by `make`.  This will generate a file called
`libpdal_plugin_reader_myreader.dylib`.

Put this dylib file into the directory pointed to by `PDAL_DRIVER_PATH`, and
then when you run `pdal --drivers`, you should see an entry for
readers.myreader.

To test the reader, we will put it into a pipeline and output a text file.

Please download the [pipeline-myreader.json] and [test-reader-input.txt] files.

In the directory with those two files, run
`pdal pipeline pipeline-myreader.json`.  You should have an output file
called `output.txt`, which will have the same data as in the input file,
except in a CSV style format, and with the Z values scaled by .001.

## Streaming Reader

Streaming points from a cloud can be accomplished via creating a custom writer
class that will query the file reader. An example of this, which also shows all
the member functions that are needed for a writer, is in
`examples/reading-streamer`.

## Fine-grained Streaming Control

Normally PDAL expects that the points will be streamed from a file without any
interruption, and be consumed as they arrive. An example showing how to
pause/resume streaming points is in `examples/batch-streamer`.

This example also shows how to use a callback, rather than creating a full
writer class. All the variables that must be shared are global.

[pipeline-myreader.json]: https://github.com/PDAL/PDAL/blob/master/examples/writing-reader/pipeline-myreader.json?raw=true
[test-reader-input.txt]: https://github.com/PDAL/PDAL/blob/master/examples/writing-reader/test-reader-input.txt?raw=true
