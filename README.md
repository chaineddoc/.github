# ChainedDocument

**ChainedDocument** is a C# document and tabular-data library designed
to work with delimited text files as structured, editable documents. Its
defining capability is **document chaining**: multiple physical files
can be linked together and accessed as one logical document.

Instead of requiring a large dataset to live in a single file,
ChainedDocument can represent it as a sequence of connected pieces. Each
piece remains a separate physical resource, while the library provides
document-level operations for reading, counting, navigating, and
modifying the combined data.

The library also includes configurable text parsing, row/column
transformations, labels, change tracking through filters, collections of
documents, and matrix-oriented views.

> **Project scope:** This README describes the original C# source in
> this repository. It does not describe the separate JavaScript port or
> imply that every external dependency is included in this source
> package.

## Contents

-   [Key features](#key-features)
-   [Conceptual model](#conceptual-model)
-   [Project structure](#project-structure)
-   [Core components](#core-components)
-   [Getting started](#getting-started)
-   [Working with chained documents](#working-with-chained-documents)
-   [Reading and writing data](#reading-and-writing-data)
-   [Labels and column metadata](#labels-and-column-metadata)
-   [Matrix documents](#matrix-documents)
-   [Connectors and text formats](#connectors-and-text-formats)
-   [Change filters](#change-filters)
-   [Events and operation monitoring](#events-and-operation-monitoring)
-   [Configuration](#configuration)
-   [Dependencies and compatibility](#dependencies-and-compatibility)
-   [Important considerations](#important-considerations)
-   [License](#license)

## Key features

-   **Chained files:** connect multiple physical files into a single
    logical document.
-   **Piece-level and document-level access:** work with an individual
    file or navigate the complete chain.
-   **Delimited-text processing:** read and write lines, rows, columns,
    and individual fields.
-   **Configurable delimiters:** configure field separators, line
    separators, quote characters, and escape characters.
-   **Quoted content handling:** detect quoted fields and preserve line
    breaks embedded in quoted content through the connector
    implementations.
-   **Flexible data construction:** create documents from paths, text,
    lines, cell collections, `DataTable`, `DataSet`, and XML inputs
    where supported by the constructors.
-   **Column and row labels:** expose labels, resolve label indexes, and
    work with inferred or forced labels.
-   **Change filters:** represent modifications and deletions as indexed
    filter operations.
-   **Matrix views:** interpret a document as a two-dimensional or
    three-dimensional data structure.
-   **Document collections:** group documents and append incoming
    documents to an existing chain by key.
-   **Events and counting hooks:** observe document activity, changes,
    and counting operations.
-   **Text and Excel-oriented connectors:** provide connector
    implementations for plain text and Excel-oriented file handling.

## Conceptual model

### A document is a chain of pieces

A `ChainedDocument` can represent one physical file or a linked sequence
of files.

Each piece can have:

-   A **back piece**: the preceding document in the chain.
-   A **fore piece**: the following document in the chain.

The chain can be traversed from its first piece to its last piece.
Document-level operations can then work across the connected pieces,
while piece-level operations target a specific physical file.

For example, imagine a dataset divided into three files:

``` text
employees-01.tsv
employees-02.tsv
employees-03.tsv
```

These files can be connected into one logical document:

``` text
employees
    |
    +-- employees-01.tsv
    |
    +-- employees-02.tsv
    |
    +-- employees-03.tsv
```

The logical document exposes the combined sequence of records without
requiring the application to first merge all files into one physical
file.

### Logical document versus physical piece

The distinction between a document and a piece is important:

  Concept       Meaning
  ------------- -----------------------------------------------------
  Document      The logical data structure represented by the chain
  Piece         One physical file/resource in that chain
  Back piece    The previous piece
  Fore piece    The next piece
  First piece   The first physical piece in the chain
  Last piece    The final physical piece in the chain

The API exposes properties such as `FirstPiece`, `LastPiece`,
`BackPiece`, `ForePiece`, `Chain`, `ForeChain`, `BackChain`, `Length`,
and `PieceIndex` to navigate this structure.

## Project structure

The source is organized into the following components:

``` text
IO/
├── ChainedDocument.cs
├── ChainedDocumentCollection.cs
├── MatrixDocument.cs
│
├── Connector/
│   ├── ConnectorBase.cs
│   ├── PlainTextFile.cs
│   └── ExcelFile.cs
│
├── Filter/
│   ├── FilterCollection.cs
│   └── FilterItem.cs
│
└── Matrices/
    ├── TextMatrixDocument.cs
    ├── MathMatrixDocument.cs
    └── ForceMathMatrixDocument.cs
```

### Component overview

  -----------------------------------------------------------------------
  Component                           Responsibility
  ----------------------------------- -----------------------------------
  `ChainedDocument`                   Main document abstraction, file
                                      chaining, data access, labels,
                                      counting, and editing

  `ChainedDocumentCollection`         Dictionary-based collection of
                                      documents with chain-aware `Add`
                                      behavior

  `ConnectorBase`                     Base abstraction for reading,
                                      writing, and converting document
                                      content

  `PlainTextFile`                     Plain-text file connector

  `ExcelFile`                         Excel-oriented connector
                                      implementation

  `FilterItem`                        Describes an indexed document
                                      change

  `FilterCollection`                  Stores and applies change filters

  `MatrixDocument<T>`                 Generic multidimensional view over
                                      a `ChainedDocument`

  `TextMatrixDocument`                String-based matrix representation

  `MathMatrixDocument`                Numeric matrix representation

  `ForceMathMatrixDocument`           Numeric matrix representation with
                                      forced numeric conversion behavior
  -----------------------------------------------------------------------

## Core components

### `ChainedDocument`

Namespace: `MiMFa.Model.IO`

`ChainedDocument` is the central class. It implements
`IEnumerable<IEnumerable<string>>` and provides the main interface for
working with structured text documents.

Its responsibilities include:

-   Creating and opening document resources.
-   Connecting and disconnecting document pieces.
-   Reading and writing lines, rows, columns, and cells.
-   Counting rows, fields, and pieces.
-   Navigating the chain and accessing individual pieces.
-   Managing row and column labels.
-   Configuring separators, quote characters, and escape characters.
-   Monitoring changes and counting operations through events.
-   Providing document-level and piece-level operations.

The class includes constructors for common data sources, including file
paths, text, line sequences, cell sequences, `DataTable`, `DataSet`, and
XML document inputs. Refer to the constructor definitions in
`ChainedDocument.cs` for the exact overloads available in the version
you are using.

### `ChainedDocumentCollection`

Namespace: `MiMFa.Model.IO`

`ChainedDocumentCollection` extends
`Dictionary<string, ChainedDocument>`.

Its `Add` methods have chain-aware behavior: when a key already exists,
the incoming document is appended to the existing document's chain
rather than simply replacing the existing entry.

This is useful when documents are grouped by a common key, such as a
directory, category, or other application-defined identifier.

### `ConnectorBase`

Namespace: `MiMFa.Model.IO.Connector`

`ConnectorBase` defines the common interface and conversion logic used
by file connectors.

It provides operations for reading and writing:

-   Text
-   Lines
-   Rows
-   Columns
-   Warps (the connector's field-oriented representation)

It also contains the parsing configuration, including:

-   `Encoding`
-   `WarpsSplitter` and `WarpsSplitters`
-   `LinesSplitter` and `LinesSplitters`
-   `QuoteChar` and `QuoteChars`
-   `MetaChar`
-   `MetaLineFeedChar`
-   `MetaCarriageReturnChar`
-   `MetaLineBreakChar`

The connector converts between text, lines, rows, columns, and field
sequences. Its virtual methods allow specialized connectors to implement
storage-specific behavior.

### `PlainTextFile`

`PlainTextFile` is the plain-text connector. It uses the MiMFa service
layer for file operations and path handling.

It supports reading and writing text and lines, as well as appending
text and lines. Its line-reading implementation also accounts for quoted
content that spans physical lines.

### `ExcelFile`

`ExcelFile` is an Excel-oriented connector implementation. It uses the
same connector abstraction and provides text/line operations for its
configured resource.

The source package references Microsoft Office Interop types elsewhere
in the project. Excel integration therefore depends on the appropriate
Office/Interop environment and should not be assumed to be a
self-contained, cross-platform Excel reader.

## Getting started

The following examples illustrate the intended API. They assume that the
original project and its external MiMFa dependencies have been
referenced by your application.

### Create a document from a file

``` csharp
using MiMFa.Model.IO;

var document = new ChainedDocument(@"C:\Data\employees.tsv");
```

The constructor and connector configuration determine how a path is
interpreted and whether a resource is created when it does not already
exist. Check the relevant constructor implementation before relying on a
particular creation behavior.

### Create a document from rows

``` csharp
using MiMFa.Model.IO;

var rows = new[]
{
    new[] { "Id", "Name", "Department" },
    new[] { "1", "Ada Lovelace", "Research" },
    new[] { "2", "Grace Hopper", "Engineering" }
};

var document = new ChainedDocument(
    @"C:\Data\employees.tsv",
    rows.Select(row => row.AsEnumerable()),
    "\t",
    Environment.NewLine
);
```

### Read rows

``` csharp
foreach (var row in document.ReadRows())
{
    Console.WriteLine(string.Join(" | ", row));
}
```

### Read lines or text

``` csharp
IEnumerable<string> lines = document.ReadLines();
string content = document.ReadText();
```

### Write rows

``` csharp
document.WriteRows(new[]
{
    new[] { "Id", "Name" },
    new[] { "1", "Ada Lovelace" },
    new[] { "2", "Grace Hopper" }
}.Select(row => row.AsEnumerable()));
```

The examples demonstrate the public API shape; exact overload resolution
can depend on the collection types used by the calling code.

## Working with chained documents

A chain can be built by appending a document as a fore piece:

``` csharp
var first = new ChainedDocument(@"C:\Data\part-01.tsv");
first.AppendForePiece(@"C:\Data\part-02.tsv");
first.AppendForePiece(@"C:\Data\part-03.tsv");
```

The resulting object represents the connected sequence. The chain can be
inspected using:

``` csharp
Console.WriteLine(first.Length);
Console.WriteLine(first.FirstPiece.Path);
Console.WriteLine(first.LastPiece.Path);

foreach (var piece in first.GetChain())
{
    Console.WriteLine(piece.Path);
}
```

The source also provides methods for setting or prepending back/fore
pieces, breaking a sequence, and accessing a piece by its position. See
`ChainedDocument.cs` for the complete set of overloads and their exact
semantics.

### Piece-level and document-level operations

The API distinguishes operations on the complete logical document from
operations on a particular piece. Methods prefixed with `Piece`
generally target the current physical piece, while document-level
methods can traverse the chain.

This distinction is useful when processing large datasets in separate
files or when a particular segment must be edited independently.

## Reading and writing data

The connector abstraction treats delimited content as a hierarchy:

``` text
Text
 └── Lines
      └── Rows
           └── Fields
```

The API also exposes column-oriented and field-oriented conversions.

Common operations include:

  Operation              Representative API
  ---------------------- ---------------------
  Read all text          `ReadText()`
  Read lines             `ReadLines()`
  Read rows              `ReadRows()`
  Read columns           `ReadColumns()`
  Read field sequences   `ReadWarps()`
  Write text             `WriteText(...)`
  Write lines            `WriteLines(...)`
  Write rows             `WriteRows(...)`
  Write columns          `WriteColumns(...)`
  Append lines           `AppendLines(...)`
  Append text            `AppendText(...)`
  Prepend lines          `PrependLines(...)`
  Prepend text           `PrependText(...)`

The connector also exposes conversion helpers such as `LineToRow`,
`LinesToRows`, `TextToLines`, `TextToRows`, and their corresponding
reverse transformations.

### Counting and dimensions

`ChainedDocument` exposes counts and size-related properties, including:

-   `RowsCount` and `LinesCount`
-   `ColumnsCount` and `WarpsCount`
-   `CellsCount`
-   `PieceLinesCount` and `PieceWarpsCount`
-   `Length` and piece/sequence length properties

Some count operations can involve scanning document pieces. The
implementation also exposes counting events and quick-count
functionality. Applications should consider the cost of counting large
or changing files and should not assume every count is a constant-time
operation.

## Labels and column metadata

ChainedDocument supports row and column labels, including labels
associated with an individual piece and labels resolved across a chain.

Relevant properties and methods include:

-   `HasColumnsLabels` and `HasRowsLabels`
-   `ColumnsLabels` and `RowsLabels`
-   `ForceColumnsLabels` and `ForceRowsLabels`
-   `GetColumnLabel(...)` and `GetRowLabel(...)`
-   `GetColumnIndex(...)` and `GetRowIndex(...)`
-   `GetForceColumnIndex(...)` and `GetForceRowIndex(...)`
-   `ColumnsLabelsTypes` and `GetColumnLabelType(...)`

The API also includes label-index suggestion and column-type suggestion
methods. These can help applications inspect data and infer likely
header positions or column types.

Label behavior is configurable. For example, `FreeColumnsLabels` and
`FreeRowsLabels` affect how labels are treated, while force-label
properties expose generated or forced label sequences.

## Matrix documents

`MatrixDocument<T>` provides a multidimensional view over a
`ChainedDocument`. It allows the document's data to be accessed as a
line, plane, or sequence of planes.

The matrix dimensions are exposed as:

-   `Width`
-   `Height`
-   `Depth`
-   `Number`
-   `Space`

The matrix API includes:

-   `Coplanar` --- the complete three-dimensional representation.
-   `Plane` --- a two-dimensional view.
-   `XPlane`, `YPlane`, and `ZPlane` --- plane-oriented projections.
-   `Line`, `XLine`, `YLine`, and `ZLine` --- line-oriented views.
-   `XYLine`, `XZLine`, and `YZLine` --- axis-oriented line views.
-   `XYZ(...)` --- access to an individual value.

The generic matrix class accepts conversion functions for converting
between stored strings and typed values.

### Specialized matrix types

  ---------------------------------------------------------------------------
  Type                        Value type              Purpose
  --------------------------- ----------------------- -----------------------
  `TextMatrixDocument`        `string`                Text matrix

  `MathMatrixDocument`        `double`                Numeric matrix

  `ForceMathMatrixDocument`   `double`                Numeric matrix with
                                                      forced conversion
                                                      behavior
  ---------------------------------------------------------------------------

Example:

``` csharp
using MiMFa.Model.IO;
using MiMFa.Model.IO.Matrices;

var source = new ChainedDocument(@"C:\Data\matrix.tsv");
var matrix = new TextMatrixDocument(source);

// Read a value using the matrix indexer.
// The indexer order is [z, y, x].
string value = matrix[0, 0, 0];

// Read matrix dimensions.
Console.WriteLine(matrix.Width);
Console.WriteLine(matrix.Height);
Console.WriteLine(matrix.Depth);
```

The matrix indexer uses `[z, y, x]` ordering, while the named `Get` and
`Set` methods accept coordinates in `(x, y, z)` order. Pay attention to
this difference when using both forms.

## Connectors and text formats

The connector layer separates the document model from the physical
representation.

### Default configuration

The base connector initializes with:

-   UTF-8 encoding.
-   A tab character as the default field separator.
-   Environment-dependent line separators, with `\n` also recognized by
    the default line-splitter configuration.
-   A double quote (`"`) as the default quote character.
-   A backslash (`\`) as the default metadata/escape character.

These values can be changed through the connector or the corresponding
document properties.

### Quoting and escaping

The connector maintains quote and separator detectors and provides
`EscapeChars` and `UnescapeChars` methods. Its parsing logic uses the
configured quote and metadata characters to distinguish separators and
line breaks that are part of field content from structural delimiters.

For exact behavior with unusual delimiters, nested quote patterns, or
malformed input, consult and test the relevant connector implementation.

## Change filters

The `Filter` namespace contains `FilterItem` and `FilterCollection`.

A filter item represents a change associated with an index or range. The
filter model includes change types such as insertion, deletion, and
modification through the `TableChangeMode` type used by the source.

`FilterCollection` provides operations to:

-   Apply filters to a sequence.
-   Add a filter while adjusting indexes.
-   Add or replace repeated filters.
-   Add multiple filters.
-   Compress redundant filter entries.
-   Find filters that apply to a given index.

The filter implementation is useful for representing edits without
necessarily rewriting the entire logical dataset for every change. Its
exact integration with document editing is defined by the original
implementation; it should not be treated as a general-purpose database
transaction log.

## Events and operation monitoring

`ChainedDocument` exposes events and handler registration methods for
observing document activity.

The source includes hooks for:

-   General activity (`Acted`)
-   Document changes (`Changed`)
-   Line changes (`LinesChanged`)
-   Field/warp changes (`WarpsChanged`)
-   Piece-line counting (`PieceLinesCounting`, `PieceLinesCounted`)
-   Line counting (`LinesCounting`, `LinesCounted`)
-   Field/warp counting (`WarpsCounting`, `WarpsCounted`)
-   Path changes (`PathChanged`)

These hooks can be used by a host application to monitor operations,
react to changes, or observe counting progress. Review the event
delegate signatures and return-value behavior before using handlers to
cancel or influence an operation.

## Configuration

The following are representative configuration properties available
through the document/connector layer:

  Property           Description
  ------------------ ----------------------------------------
  `Encoding`         Text encoding
  `WarpsSplitter`    Primary field separator
  `WarpsSplitters`   Supported field separators
  `LinesSplitter`    Primary line separator
  `LinesSplitters`   Supported line separators
  `QuoteChar`        Primary quote character
  `QuoteChars`       Supported quote characters
  `MetaChar`         Escape/metadata character
  `DefaultConfig`    Connector default-configuration flag
  `IsAutoSave`       Document auto-save setting
  `OptimizedTypes`   Type optimization setting
  `TypesStandard`    Standard used for column type metadata

Some configuration is exposed on `ChainedDocument` as a convenience
wrapper over its connector. Consult the source for defaults and the
effect of changing configuration after a document has already been
opened or parsed.

## Dependencies and compatibility

The supplied source files reference types outside this directory. In
particular, they use the MiMFa service/general libraries and several
framework or platform APIs.

Dependencies visible in the source include:

-   `MiMFa.Service`
-   `MiMFa.General`
-   .NET APIs such as `System.IO`, `System.Data`, `System.Xml`,
    `System.Text`, and LINQ
-   Microsoft Office Interop for Excel-related integration
-   Visual Basic computer-information APIs
-   Windows-related namespaces referenced by the source

Consequently:

1.  **This source folder is not, by itself, a complete standalone
    project.** The required MiMFa dependencies and project configuration
    must also be available.
2.  **Excel support has an Office/Interop dependency.** Availability and
    deployment depend on the target Windows/Office environment.
3.  **Cross-platform compatibility should not be assumed.** Some
    referenced APIs are Windows-specific or tied to the surrounding
    MiMFa application environment.
4.  **Target-framework compatibility must be checked.** The original
    source should be built and tested against the framework and
    dependency versions used by the consuming application.

No NuGet package name, target framework, or license is asserted here
because those details are not established by the supplied source files.
Use the repository's project metadata and distribution terms as the
authoritative references.

## Important considerations

-   **Chaining is logical, not necessarily physical merging.**
    Connecting pieces does not mean their contents are automatically
    copied into one file.
-   **Choose the correct operation scope.** A piece-level operation and
    a document-level operation may have different effects.
-   **Be deliberate with separators and encoding.** Files created by
    different applications may use different encodings, quoting rules,
    or newline conventions.
-   **Validate input data.** Delimited text can contain malformed
    quoting, inconsistent field counts, or unexpected line breaks.
-   **Test large datasets.** Counting, enumeration, and transformations
    may read data from multiple pieces. Measure performance with the
    actual storage and workload.
-   **Review external dependencies before deployment.** The Excel
    connector and platform-specific APIs may constrain where the
    original implementation can run.
-   **Check the original source for exact semantics.** This README is an
    overview and usage guide, not a replacement for API documentation or
    tests.

## License

No license information was present in the supplied source package.
Before publishing or redistributing this project, add the license that
applies to the original work and ensure that all included dependencies
and third-party integrations are compatible with it.

------------------------------------------------------------------------

**ChainedDocument** --- a chained, file-oriented document model for
structured text data.
