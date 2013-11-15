---
layout: post
title: Data Flow Operations in KijiExpress
categories: [userguides, express, devel]
tags : [express-ug]
version: devel
order : 7
description: Data Flow Operations in KijiExpress.
---

Here are some common tasks done using Kiji and KijiExpress:

* Write an “importer” for each Kiji table you want to fill with source data.
* Perform some logic to manipulate the source data.
* Write the resulting data into Kiji tables.


#### Writing an Importer for the Source Data

KijiSchema provides a stock importer you can use to import JSON data.

If the stock importer isn’t appropriate for what you are doing (especially when you first
start out and are looking for something with a little less overhead), you can write your
own importer using Scala. The contents of the importer are as follows:

* Identify the source data location
* Identify the target Kiji table
* Define any functions you want to operate on the data before it is written to the target
* Define the mapping between the source data and the target columns

The fastest way to write an importer may be to copy one of the Scala importer files from a
Kiji tutorial project and modify it to meet your needs. Find an example here:

    ${KIJI_HOME}/examples/express-music/src/main/scala/org/kiji/express/music/SongMetadataimporter.scala

For a walkthrough of Scala syntax, see [Writing Basic KijiExpress Jobs](../basic-jobs).

### Processing Data

### Mapping Data

With data streaming in the processing pipeline, the KijiExpress operation can begin to
manipulate the data. The most common operation to apply to data is to restructure the
arrangement of the data using a map operation. Scala provides more than one map-type operation;
we'll go through the four most common here:

* [Map](#map)
* [FlatMap](#flatmap)
* [MapTo](#mapto)
* [FlatMapTo](#flatmapto)

#### Map

Map operations in Scala use the following syntax:

    map('existing_field -> 'additional_field) { mapping_function }

This syntax is simplified: it shows single field mapped one-to-one to another single field.
A map function can operation on any number of fields at once, and the mapping doesn't
have to be one-to-one.

The mapping function can be something defined elsewhere (in an included library or elsewhere
in the file) or written out between the curly braces. The mapping function needs to be told
the types of the input it operates on.

The output of a map operation on the data stream always augments the input stream: the
output is a tuple that includes the entire input stream plus whatever additional target
fields are specified.

>The keyword `map` also appears as a constructor for a map datatype. For example, `Map`
>appears as a KijiInput parameter, it maps columns from the table into tuples in the
>output data stream. See KijiSources ?link.

Here are some examples of map functions:

##### Parsing row content into tuple fields

This example takes the line field (output from `TextLine`) and uses a mapping function
`parseJson` to transform the JSON record into tuple fields.

Notice that the map target includes multiple fields: the `parseJson` function has to
produce these fields as its output.

    .map('line -> ('userId, 'playTime, 'songId)) { parseJson}

The map function does not need to indicate the types of the output fields because they
are specified in the definition of `parseJson`:

    def parseJson(json: String): (String, Long, String) = {
        ...
    }

The anonymous map function augments the input data stream with the new material specified
in the target; each "row" of the the output data stream is a tuple that consists of four
fields: `line`, `userId`, `playTime`, and `songId`.

##### Inserting a value as an Entity ID

This example takes the values of one field, transforms them by applying the mapping
function, then puts the result in another field.

    .map('firstSong -> 'entityId) {
        firstSong: String => EntityId(firstSong) }

This example has the following points:

* Data from one field is mapped to another single field.

* The mapping function specification includes the data type of the source field
(`firstSong: String`).

* The operation that the mapping function performs is defined by EntityId, which is a
KijiExpress method that creates an entity ID from the objects passed to it.
(`EntityId(firstSong)`).

* It augments the input stream so that the output stream includes the entire content of
the input stream augmented with the new field entityId. The result is a stream that's
ready to be written to a Kiji table.

##### Processing Kiji Table Columns

A map statement can include the transformation in line instead of specifying a function
defined elsewhere. This example takes a column from a Kiji table and returns only the most
recent value of the column. ?link to something about how kiji tables are structured.
It uses the fact that the content of the column can be manipulated as a `KijiSlice`:
Kiji tables can hold many timestamped values (versions) for each column; accessing this
data as a `KijiSlice` allows you to pull a specific version for the column by index or
timestamp.

    .map('trackPlays -> 'lastTrackPlayed) {
        slice: KijiSlice[String] => slice.getFirstValue()}
    .mapTo(('entityId, 'name, 'desc) -> ('id, 'name, 'desc)) {
        cols: (EntityId, KijiSlice[String], KijiSlice[String]) =>
          val (entityId, name, desc) = cols
          (entityId(0), name.getFirstValue(), desc.getFirstValue())

This example has the same basic structure as previous examples:

* Data from one field is mapped to another single field.
* The mapping function includes the data type of the source field, but this time, the
source is not called out by its name `trackPlays` but instead the statement takes advantage
of the fact that the source field is a Kiji table column and that it is the only source
field identified. The type description defines a new object "slice" of type `KijiSlice`
where the underlying content of the slice is of type String.

?What would happen if there were more than one source fields?
?Anything else assumed or hidden here?

* The mapping function transforms this input object "slice" by applying the `KijiSlice`
method `getFirstValue()`. `KijiSlice` provides many similar methods that make operations
across all of the versions of the column value very easy; some of them are `avg`,
`getLastValue`, `groupBy`, `max`, `min`, `orderBy`, `size`, `sum`. The whole list is here:
?link to KijiSlice API, or more in depth discussion of KijiSlice.

* It augments the input stream so that the output stream includes the entire content of
the input stream (`trackPlays`) augmented with the new field (`lastTrackPlayed`).

#### FlatMap

FlatMap operations in Scala use the same syntax as map operations:

    .flatMap('existing_field -> 'additional_field) { mapping_function }

Just like a map operation, you can specify multiple existing or additional fields and the
mapping function can be defined in-line or elsewhere in the file or in an included library.

The `flatMap` operation performs the map to generate a list of new values to fill the
additional fields, then flattens that list into individual tuples. For each tuple of input,
the `flatMap` generates a tuple for each value produced by the mapping function. Each output
tuple includes the entire input tuple plus the additional field value.
Here are some examples of `flatMap` functions:

##### Document to Words

The quintessential use of `flatMap` makes an index of values such as turning a document
into a list of words. The `flapMap` statement takes a string, splits it into "words"
delimited by spaces, and then produces a tuple for each word is as follows:

    .flatMap('doc_content -> 'word) {
        doc_content : String => doc_content.split("\s+")
        }

As with a `map` operation, the `flatMap` operation has the following characteristics:

* Data from one field is mapped to another single field.

* The mapping function includes the data type of the source field: `(doc_content : String)`.

* Scala includes a number of methods that can be applied against a string value that are
defined for the Scala class "StringLike". One is `split`; others include `append`, `capitalize`,
`compare`, `count`, `distinct`, `isEmpty`, `last`, `sortBy`, and `toSet`. In this case, the
`split` method takes a character or a regular expression; this example uses the regular expression
`\s+` to specify one or more spaces.

* The mapping function splits the input into words; if this were a `map` function, the
output would be the input tuple with an additional field that included a list of all the
words in the `doc_content` field of the original tuple. Because it is a `flatMap` function,
the output is a tuple for each word in `doc_content`. Each output tuple contains the fields
from the input tuple with an additional field `word` with one of the words from `doc_content`.
?Is the list ordered in any way?

##### FlatMap With Multiple Fields in the Output

The Document to Words example is simple in that it pivots on a single field value; what
happens when there's more payload in the output?

The following example includes multiple "additional fields" grouped within parentheses:

    .flatMap('playlist -> ('firstSong, 'songId)) { bigrams }

The interesting pieces of this statement are:

* The fields added to the output data stream are grouped in parentheses to indicate that
they together compose the new value that is the seed of the new output tuple.

* The mapping function `bigrams` converts the list of songs from playlist (organized as a
set of versions in a single column of a Kiji table, see `trackPlays` in Processing Kiji
Table Columns) into a collection of tuples, each tuple including two song names of songs
that were played one after the other. The input in this case is a stream of tuples from a
Kiji table. The field `playlist` has been produced as an output of a `KijiInput` source
and can be manipulated as a `KijiSlice`:

![KijiSlice][express_playlist]

[express_playlist]: ../../../../assets/images/express_playlist.png

The mapping function bigram turns playlist into a collection of tuples:

![Bigram][express_bigram]

[express_bigram]: ../../../../assets/images/express_bigram.png

The output of the `flatMap` is a tuple for each value of the collection of first song and
its ID; the tuple includes the original playlist values: ?how do the song IDs come back?

![FlatMap][express_flatmap]

[express_flatmap]: ../../../../assets/images/express_flatmap.png

The details of how to define the mapping function to generate bigrams is described in
?tutorial_link?

#### MapTo

MapTo operations perform a map operation followed by a "project" operation; using a
`mapTo` function is more efficient that performing the two operations separately. The
project operation drops the original input fields and retains only the fields indicated
in the map operation target field list.

    .mapTo((existing_field_list) -> (output_field_list)) { mapping_function }

After this oepration, the only fields of the tuples in the pipe are those in output_field_list.


#### FlatMapTo

FlatMapTo is similar to MapTo: it performs a flatMap, but keeps only the output fields. This snippet
comes from the PlayCounter in the KijiExpress music tutorial:

    .flatMapTo('playlist -> 'song) {
        slice : Seq[FlowCell[[String] => slice.cells.map {cell => cell.datum} }

After this step in the flow, the only field in the pipe is 'song.

### Building from Examples

There are a number of examples provided in the KijiExpress tutorial. The following list
indexes some of the functionality shown in the tutorial examples with the specific
example file:

| Functionality | Example |
| ------------- | ------- |
| Pass arguments from the command line | all |
| Read data from a Kiji table | all |
| Call a function to operate on the pipeline data | all |
| Create an EntityId from a column in the source | SongRecommender.scala |
| Read a column with multiple values and produce a tuple | SongRecommender.scala |
| Filters the pipeline input on specific columns (project) | SongRecommender.scala |
| Extract a single data value from a column | SongRecommender.scala |
| Join data from two pipelines | SongRecommender.scala |
| Breaks a list into individual values | SongPlayCounter.scala |
| Groups values, counts instances | SongPlayCounter.scala |
| Create a sorted key-value store | TopNextSongs.scala |
| Fill an Avro record | TopNextSongs.scala |
| Write data to a Kiji table | all |
| Write data to a HFDS file | SongPlayCounter.scala |


These Scala files are here:

    ${KIJI_HOME}/examples/express-music/src/main/scala/org/kiji/express/music/

