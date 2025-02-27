# pprof

pprof is a tool for visualization and analysis of profiling data.

pprof reads a collection of profiling samples in profile.proto format and
generates reports to visualize and help analyze the data. It can generate both
text and graphical reports (through the use of the dot visualization package).

profile.proto is a protocol buffer that describes a set of callstacks
and symbolization information. A common usage is to represent a set of
sampled callstacks from statistical profiling. The format is
described on the proto/profile.proto file. For details on protocol
buffers, see https://developers.google.com/protocol-buffers

Profiles can be read from a local file, or over http. Multiple
profiles of the same type can be aggregated or compared.

If the profile samples contain machine addresses, pprof can symbolize
them through the use of the native binutils tools (addr2line and nm).

# pprof profiles

pprof operates on data in the profile.proto format. Each profile is a collection
of samples, where each sample is associated to a point in a location hierarchy,
one or more numeric values, and a set of labels. Often these profiles represents
data collected through statistical sampling of a program, so each sample
describes a program call stack and a number or value of samples collected at a
location. pprof is agnostic to the profile semantics, so other uses are
possible. The interpretation of the reports generated by pprof depends on the
semantics defined by the source of the profile.

# Usage modes

There are few different ways of using `pprof`.

## Report generation

If a report format is requested on the command line:

    pprof <format> [options] source

pprof will generate a report in the specified format and exit.
Formats can be either text, or graphical. See below for details about
supported formats, options, and sources.

## Interactive terminal use

Without a format specifier:

    pprof [options] source

pprof will start an interactive shell in which the user can type
commands.  Type `help` to get online help.

## Web interface

If a host:port is specified on the command line:

    pprof -http=[host]:[port] [options] source

pprof will start serving HTTP requests on the specified port.  Visit
the HTTP url corresponding to the port (typically `http://<host>:<port>/`)
in a browser to see the interface.

# Details

The objective of pprof is to generate a report for a profile. The report is
generated from a location hierarchy, which is reconstructed from the profile
samples. Each location contains two values:

* *flat*: the value of the location itself.
* *cum*: the value of the location plus all its descendants.

Samples that include a location multiple times (e.g. for recursive functions)
are counted only once per location.

## Options

*options* configure the contents of a report. Each option has a value,
which can be boolean, numeric, or strings. While only one format can
be specified, most options can be selected independently of each
other.

Some common pprof options are:

* **-flat** [default], **-cum**: Sort entries based on their flat or cumulative
  value respectively, on text reports.
* **-functions** [default], **-filefunctions**, **-files**, **-lines**,
  **-addresses**: Generate the report using the specified granularity.
* **-noinlines**: Attribute inlined functions to their first out-of-line caller.
  For example, a command like `pprof -list foo -noinlines profile.pb.gz` can be
  used to produce the annotated source listing attributing the metrics in the
  inlined functions to the out-of-line calling line.
* **-nodecount= _int_:** Maximum number of entries in the report. pprof will
  only print this many entries and will use heuristics to select which entries
  to trim.
* **-focus= _regex_:** Only include samples that include a report entry matching
  *regex*.
* **-ignore= _regex_:** Do not include samples that include a report entry
  matching *regex*.
* **-show\_from= _regex_:** Do not show entries above the first one that
  matches *regex*.
* **-show= _regex_:** Only show entries that match *regex*.
* **-hide= _regex_:** Do not show entries that match *regex*.

Each sample in a profile may include multiple values, representing different
entities associated to the sample. pprof reports include a single sample value,
which by convention is the last one specified in the report. The `sample_index=`
option selects which value to use, and can be set to a number (from 0 to the
number of values - 1) or the name of the sample value.

Sample values are numeric values associated to a unit. If pprof can recognize
these units, it will attempt to scale the values to a suitable unit for
visualization. The `unit=` option will force the use of a specific unit. For
example, `unit=sec` will force any time values to be reported in
seconds. pprof recognizes most common time and memory size units.

## Tags

Samples in a profile may have tags. These tags have a name and a value. The
value can be either numeric or a string; the numeric values can be associated
with a unit. Tags are used as additional dimensions that the sample values can
be broken by. The most common use of tags is selecting samples from a profile
based on the tag values. pprof also supports tags at the visualization time.

### Tag filtering

The `-tagfocus` option is the most used option for selecting data in a profile
based on tag values. It has the syntax of **-tagfocus=_regex_** or
**-tagfocus=_range_:** which will restrict the data to samples with tags matched
by regexp or in range. The `-tagignore` option has the identical syntax and can
be used to filter out the samples that have matching tags. If both `-tagignore`
and `-tagfocus` are specified and match a given sample, then the sample will be
discarded.

When using `-tagfocus=regex` and `-tagignore=regex`, the regex will be compared
to each value associated with each tag. If one specifies a value
like `regex1,regex2`, then only samples with a tag value matching `regex1`
and a tag value matching `regex2` will be kept.

In addition to being able to filter on tag values, one can specify the name of
the tag which a certain value must be associated with using the notation
`-tagfocus=tagName=value`. Here, the `tagName` must match the tag's name
exactly, and the value can be either a regex or a range. If one specifies
a value like `regex1,regex2`, then samples with a tag value (paired with the
specified tag name) matching either `regex1` or matching `regex2` will match.

Here are examples explaining how `-tagfocus` can be used:

* `-tagfocus 128kb:512kb` accepts a sample iff it has any numeric tag with
  memory value in the specified range.
* `-tagfocus mytag=128kb:512kb` accepts a sample iff it has a numeric tag
  `mytag` with memory value in the specified range. There isn't a way to say
   `-tagfocus mytag=128kb:512kb,16kb:32kb`
   or `-tagfocus mytag=128kb:512kb,mytag2=128kb:512kb`. Just single value or
   range for numeric tags.
* `-tagfocus someregex` accepts a sample iff it has any string tag with
  `tagName:tagValue` string matching specified regexp. In the future, this
  will change to accept sample iff it has any string tag with `tagValue` string
  matching specified regexp.
* `-tagfocus mytag=myvalue1,myvalue2` matches if either of the two tag values
  are present.

### Tag visualization

To list the tags and their values available in a profile use **-tags** option.
It will output the available tags and their values as well as the breakdown of
the sample value by the values of each tag.

The pprof callgraph reports, such as `-web` or raw `-dot`, will automatically
visualize the values for all tags as pseudo nodes in the graph. Use `-tagshow`
and `-taghide` options to limit what tags are displayed. The options accept a
regular expression that is matched against the tag name to show or hide it
respectively.

Options `-tagroot` and `-tagleaf` can be used to create pseudo stack frames to
the profile samples. For example, `-tagroot=mytag` will add stack frames at the
root of the profile call tree with the value of the tag for the corresponding
samples. Similarly, `-tagleaf=mytag` will add such stack frames as leaf nodes of
each sample. These options are useful when visualizing a profile in tree formats
such as the tree view in the `-http` mode web UI.

## Text reports

pprof text reports show the location hierarchy in text format.

* **-text:** Prints the location entries, one per line, including the flat and
  cum values.
* **-tree:** Prints each location entry with its predecessors and successors.
* **-peek= _regex_:** Print the location entry with all its predecessors and
  successors, without trimming any entries.
* **-traces:** Prints each sample with a location per line.

## Graphical reports

pprof can generate graphical reports on the DOT format, and convert them to
multiple formats using the graphviz package.

These reports represent the location hierarchy as a graph, with a report entry
represented as a node. Nodes are removed using heuristics to limit the size of
the graph, controlled by the *nodecount* option.

* **-dot:** Generates a report in .dot format. All other formats are generated
  from this one.
* **-svg:** Generates a report in SVG format.
* **-web:** Generates a report in SVG format on a temp file, and starts a web
  browser to view it.
* **-png, -jpg, -gif, -pdf:** Generates a report in these formats.

### Interpreting the Callgraph

* **Node Color**:
  * large positive cum values are red.
  * large negative cum values are green; negative values are most likely to
    appear during profile comparison, see [this section](#comparing-profiles)
    for details.
  * cum values close to zero are grey.

* **Node Font Size**:
  * larger font size means larger absolute flat values.
  * smaller font size means smaller absolute flat values.

* **Edge Weight**:
  * thicker edges indicate more resources were used along that path.
  * thinner edges indicate fewer resources were used along that path.

* **Edge Color**:
  * large positive values are red.
  * large negative values are green.
  * values close to zero are grey.

* **Dashed Edges**: some locations between the two connected locations were
  removed.

* **Solid Edges**: one location directly calls the other.

* **"(inline)" Edge Marker**: the call has been inlined into the caller.

Let's consider the following example graph:

![callgraph](images/callgraph.png)

* For nodes:
  * `(*Rand).Read` has a small flat value and a small cum value because the
    the font is small and the node is grey.
  * `(*compressor).deflate` has a large flat value and a large cum value because the font
    is large and the node is red.
  * `(*Writer).Flush` has a small flat value and a large cum value because the font is
    small and the node is red.

* For edges:
  * the edge between `(*Writer).Write` and `(*compressor).write`:
    * Since it is a dashed edge, some nodes were removed between those two.
    * Since it is thick and red, more resources were used in call stacks between
    those two nodes.
  * the edge between `(*Rand).Read` and `read`:
    * Since it is a dashed edge, some nodes were removed between those two.
    * Since it is thin and grey, fewer resources were used in call stacks
    between those two nodes.
  * the edge between `read` and `(*rngSource).Int63`:
    * Since it is a solid edge, there are no nodes between those two (i.e. it
      was a direct call).
    * Since it is thin and grey, fewer resources were used in call stacks
      between those two nodes.

## Annotated code

pprof can also generate reports of annotated source with samples associated to
them. For these, the source or binaries must be locally available, and the
profile must contain data with the appropriate level of detail.

pprof will look for source files on its current working directory and all its
ancestors. pprof will look for binaries on the directories specified in the
`$PPROF_BINARY_PATH` environment variable, by default `$HOME/pprof/binaries`
(`%USERPROFILE%\pprof\binaries` on Windows). It will look binaries up by name,
and if the profile includes linker build ids, it will also search for them in
a directory named as the build id.

pprof uses the binutils tools to examine and disassemble the binaries. By
default it will search for those tools in the current path, but it can also
search for them in a directory pointed to by the environment variable
`$PPROF_TOOLS`.

* **-list= _regex_:** Generates an annotated source listing for functions
  matching *regex*, with flat/cum values for each source line.
* **-disasm= _regex_:** Generates an annotated disassembly listing for
  functions matching *regex*.
* **-weblist= _regex_:** Generates a source/assembly combined annotated listing
  for functions matching *regex*, and starts a web browser to display it.

## Comparing profiles

pprof can subtract one profile from another, provided the profiles are of
compatible types (i.e. two heap profiles). pprof has two options which can be
used to specify the filename or URL for a profile to be subtracted from the
source profile:

* **-diff_base= _profile_:** useful for comparing two profiles. Percentages in
the output are relative to the total of samples in the diff base profile.

* **-base= _profile_:** useful for subtracting a cumulative profile, like a
[golang block profile](https://golang.org/doc/diagnostics.html#profiling),
from another cumulative profile collected from the same program at a later time.
When comparing cumulative profiles collected on the same program, percentages in
the output are relative to the difference between the total for the source
profile and the total for the base profile.

The **-normalize** flag can be used when a base profile is specified with either
the `-diff_base` or the `-base` option. This flag scales the source profile so
that the total of samples in the source profile is equal to the total of samples
in the base profile prior to subtracting the base profile from the source
profile. Useful for determining the relative differences between profiles, for
example, which profile has a larger percentage of CPU time used in a particular
function.

When using the **-diff_base** option, some report entries may have negative
values. If the merged profile is output as a protocol buffer, all samples in the
diff base profile will have a label with the key "pprof::base" and a value of
"true". If pprof is then used to look at the merged profile, it will behave as
if separate source and base profiles were passed in.

When using the **-base** option to subtract one cumulative profile from another
collected on the same program at a later time, percentages will be relative to
the difference between the total for the source profile and the total for
the base profile, and all values will be positive. In the general case, some
report entries may have negative values and percentages will be relative to the
total of the absolute value of all samples when aggregated at the address level.

# Fetching profiles

pprof can read profiles from a file or directly from a URL over http or https.
Its native format is a gzipped profile.proto file, but it can
also accept some legacy formats generated by
[gperftools](https://github.com/gperftools/gperftools).

When fetching from a URL handler, pprof accepts options to indicate how much to
wait for the profile.

* **-seconds= _int_:** Makes pprof request for a profile with the specified
  duration in seconds. Only makes sense for profiles based on elapsed time, such
  as CPU profiles.
* **-timeout= _int_:** Makes pprof wait for the specified timeout when
  retrieving a profile over http. If not specified, pprof will use heuristics to
  determine a reasonable timeout.

pprof also accepts options which allow a user to specify TLS certificates to
use when fetching or symbolizing a profile from a protected endpoint. For more
information about generating these certificates, see
https://docs.docker.com/engine/security/https/.

* **-tls\_cert= _/path/to/cert_:** File containing the TLS client certificate
  to be used when fetching and symbolizing profiles.
* **-tls\_key= _/path/to/key_:** File containing the TLS private key to be used
  when fetching and symbolizing profiles.
* **-tls\_ca= _/path/to/ca_:** File containing the certificate authority to be
  used when fetching and symbolizing profiles.

pprof also supports skipping verification of the server's certificate chain and
host name when collecting or symbolizing a profile. To skip this verification,
use "https+insecure" in place of "https" in the URL.

If multiple profiles are specified, pprof will fetch them all and merge
them. This is useful to combine profiles from multiple processes of a
distributed job. The profiles may be from different programs but must be
compatible (for example, CPU profiles cannot be combined with heap profiles).

## Symbolization

pprof can add symbol information to a profile that was collected only with
address information. This is useful for profiles for compiled languages, where
it may not be easy or even possible for the profile source to include function
names or source coordinates.

pprof can extract the symbol information locally by examining the binaries using
the binutils tools, or it can ask running jobs that provide a symbolization
interface.

pprof will attempt symbolizing profiles by default, and its `-symbolize` option
provides some control over symbolization:

* **-symbolize=none:** Disables any symbolization from pprof.

* **-symbolize=local:** Only attempts symbolizing the profile from local
  binaries using the binutils tools.

* **-symbolize=remote:** Only attempts to symbolize running jobs by contacting
  their symbolization handler.

For local symbolization, pprof will look for the binaries on the paths specified
by the profile, and then it will search for them on the path specified by the
environment variable `$PPROF_BINARY_PATH`. Also, the name of the main binary can
be passed directly to pprof as its first parameter, to override the name or
location of the main binary of the profile, like this:

    pprof /path/to/binary profile.pb.gz

By default pprof will attempt to demangle and simplify C++ names, to provide
readable names for C++ symbols. It will aggressively discard template and
function parameters. This can be controlled with the `-symbolize=demangle`
option. Note that for remote symbolization mangled names may not be provided by
the symbolization handler.

* **-symbolize=demangle=none:** Do not perform any demangling. Show mangled
  names if available.

* **-symbolize=demangle=full:** Demangle, but do not perform any
  simplification. Show full demangled names if available.

* **-symbolize=demangle=templates:** Demangle, and trim function parameters, but
  not template parameters.

# Web Interface

When the user requests a web interface (by supplying an `-http=[host]:[port]`
argument on the command-line), pprof starts a web server and opens a browser
window pointing at that server. The web interface provided by the server allows
the user to interactively view profile data in multiple formats.

The top of the display is a header that contains some buttons and menus.

## View

The `View` menu allows the user to switch between different visualizations of
the profile.

Top
:   Displays a tabular view of profile entries. The table can be sorted
    interactively.

Graph
:   Displays a scrollable/zoomable graph view; each function (or profile entry)
    is represented by a node and edges connect callers to callees.

[Flame Graph](#flame-graph)
:   Displays a view similar to a
    [flame graph](https://www.brendangregg.com/flamegraphs.html)
    that can show the selected node's callers and callees simultaneously.

Peek
:   Shows callers / callees per function in a simple textual forma.

Source
:   Displays source code annotated with profile information. Clicking on a
    source line can show the disassembled machine instructions for that line.

Disassemble
:   Displays disassembled machine instructions annotated with profile
    information.

## Config

The `Config` menu allows the user to save the current refinement
settings (e.g., the focus and hide list) as a named configuration. A
saved configuration can later be re-applied to reinstitue the saved
refinements. The `Config` menu contains:

**Save as ...**: shows a dialog where the user can type in a
configuration name. The current refinement settings are saved under
the specified name.

**Default**: switches back to the default view by removing all refinements.

The `Config` menu also contains an entry per named
configuration. Selecting such an entry applies that configuration. The
currently selected entry is marked with a ✓. Clicking on the 🗙 on the
right-hand side of such an entry deletes the configuration (after
prompting the user to confirm).

## Flame graph

The `Flame graph` view displays profile information as a [flame
graph](https://www.brendangregg.com/flamegraphs.html).

Boxes on this view correspond to stack frames in the profile. Caller boxes are
directly above callee boxes. The width of each box is proportional to the sum of
the sample value of profile samples where that frame was present on the call
stack. Children of a particular box are laid out left to right in decreasing
size order.

Names displayed in different boxes may have different font sizes. These size
differences are due to an attempt to fit as much of the name into the box as
possible; no other interpretation should be placed on the size.

Boxes are colored according to the name of the package in which the corresponding
function occurs. E.g., in C++ profiles all frames corresponding to `std::` functions
will be assigned the same color.

Inlining is indicated by the absence of a horizontal border between a caller and
a callee. E.g., suppose X calls Y calls Z and the call from Y to Z is inlined into
Y. There will be a black border between X and Y, but no border between Y and Z.

## TODO: cover the following issues:

*   Overall layout
*   Menu entries
*   Explanation of all the views
