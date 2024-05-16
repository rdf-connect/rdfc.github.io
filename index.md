# rdf-connect

Rdf-Connect is a pipeline runner, created to build interoperable pipelines.
These pipelines consist of multiple processors, each processor can be written in a different programming language (note currently only javacript is supported), is described with RDF and is configured with RDF.

Rdf Connnect connects different processors together using channels.
A channel is a medium that transfers bytes, each programming language can supports different channels, but when a type of channel is the same, those programming languages can be connected together.

Javacript supports the following channels:
- In-memory: plain old in memory channel, communication between javacript processors should happen using this channel.
- File: a file channel communitcates using files, reading listens for change events and emits the contents of the file as a message.
- Http: a http channel uses HTTP Post to send a message.

## Building a pipeline

Building a pipeline comes in two steps: finding the correct processors, building the pipeline.

### Choosing the correct processors

Most Rdf-Connect processors can be found on [github](https://github.com/orgs/rdf-connect/repositories).

A few promiment processors are the following:
- Creating a LDES: [sds-storage-writer-mongo](https://github.com/rdf-connect/sds-storage-writer-mongo)
- Consuming a LDES: [ldes-client](https://github.com/rdf-connect/ldes-client)
- SDS utils: [SDS-Processors](https://github.com/rdf-connect/sds-processors)
- Files and utils: [file-utils](https://github.com/rdf-connect/file-utils-processors-ts)
  - `js:GlobRead`
  - `js:Envsub`
  - `js:UnzipFile`
- Http utils: [http-utils](https://github.com/rdf-connect/http-utils-processor-ts)
- RML mapper: [RML mapper](https://github.com/rdf-connect/rml-mapper-processor-ts)


### Building the pipeline

A pipeline is a turtle file that links different processors together.
Everything is configured in RDF, but to understand what properties are exposed for each processor, you need to look for the processor definition file.
This file is often called `processor.ttl` inside the repositories.

It is required to let rdf-connect know which processors you are going to use inside the pipeline.
So each processor definition file needs to be imported in your pipeline file.
For example if you want to create a LDES, you will first install the sds-storage-writer-mongo, and import the processor definition file.

```bash
npm install @treecg/sds-storage-writer-mongo

cat > ./pipeline.ttl << EOF
> @prefix owl: <http://www.w3.org/2002/07/owl#>.
>
> <> owl:imports <./node_modules/@treecg/sds-storage-writer-mongo/processor.ttl>.
> EOF
```

The following example shows a possible configuration for that processor.
First two channels Javacript in memory channels are defined, `data/{reader|writer}` and `metadata/{reader/writer}`, they are linked together in a channel that has those components are reader and writer.
This tells the Javacript runner that they are connected.

The sds-storage-writer-mongo exposes a processor called `js:SDSIngest`. Part of the Javacript runner family and ingests SDS members.
It takes a data and a metadata reader and a mongo data endpoint.
The writer parts of the channels will be linked with some other processors inside the pipeline, probably a `js:Sdsify` from the `sds-processors`.

```turtle
<data/writer> a js:JsWriterChannel.
<data/reader> a js:JsReaderChannel.
[ ] a :Channel;
  :reader <data/reader>;
  :writer <data/writer>.
  
<metadata/writer> a js:JsWriterChannel.
<metadata/reader> a js:JsReaderChannel.
[ ] a :Channel;
  :reader <metadata/reader>;
  :writer <metadata/writer>.
  
[ ] a js:SDSIngest;
  js:dataInput <data/reader>;
  js:metadataInput <metadata/reader>;
  js:database "mongodb://127.0.0.1:27017/mumotest".
```

To understand which properties are expected you need to look into the processor definition file.
Each processor has two definition parts, a processor definition and a shacl shape.

The processor definition is only important when creating a new processor and will be covered in the next section, but note that the type is `js:JsProcess`.

The shacl shape indicates the required properties, for example the shape of `js:SDSIngest` is the following. It requires two readers and a string.
```turtle
[ ] a sh:NodeShape;
  sh:targetClass js:SDSIngest;
  sh:property [
    sh:class :ReaderChannel;
    sh:path js:dataInput;
    sh:name "Data Input Channel";
    sh:minCount 1;
    sh:maxCount 1;
  ], [
    sh:class :ReaderChannel;
    sh:path js:metadataInput;
    sh:name "Metadata Input Channel";
    sh:minCount 1;
    sh:maxCount 1;
  ], [
    sh:datatype xsd:string;
    sh:path js:database;
    sh:minCount 1;
    sh:maxCount 1;
    sh:name "Database Url";
  ].
```

Running the pipeline should be as simple as

> npx js-runner pipeline.ttl

Things might not work as expected, all errors concerning missing file extensions, esm + cjs errors and the like can be easily solved by using bun.

> bunx --bun js-runner pipeline.ttl


Enjoy!



### Creating a processor

A big part which makes rdf-connect cool is the ability to easily create new processors.
Javascript processors are only a function and some configuration.

To start, copy our template repository [template-processor](https://github.com/rdf-connect/template-processor-ts).
The provided source file is heavenly commented and provides a based structure for a logging processor that logs incoming messages before forwarding them.


The shacl shapes that define processors can be a bit tricky, here are some examples. The shapes are handled by [`rdf-lens`](https://github.com/ajuvercr/rdf-lens).

First note, the mapping between the `fno:mapping` and the `sh:property` happens with `fnom:functionParameter` and `sh:name` respectively.

Each property either has a `sh:datatype` or a `sh:class`. `sh:datatype` are primitives like `xsd:boolean`, `xsd:{integer|float|double|decimal}`, `xsd:string` or `xsd:dateTime`.
`sh:class` points to other shapes defined in the definition file, this allows to create deep objects.

```turtle
[ ] a sh:NodeShape;
  sh:targetClass <Config>;
  sh:property [
    sh:class rdfl:TypedExtract;
    sh:path ( );
    sh:name "strategy";
    sh:maxCount 1;
  ], [
    sh:datatype xsd:iri;
    sh:path ( );
    sh:name "identifier";
    sh:minCount 1;
    sh:maxCount 1;
  ].
```

This new shape defines `<Config>`. Config stores the current identifier as field `identifier` in a JSON object and field `strategy` gets an object that is defined by the provided type in the pipeline file.
For this to work, other shapes have to befined inside the processor definition file that are allowed inside the pipeline file.

Lets assume that the following shape is also defined.
```turtle
[ ] a sh:NodeShape;
  sh:targetClass js:MyEpicConfig;
  sh:property [
    sh:datatype xsd:boolean;
    sh:path js:isEpic;
    sh:name "isEpic";
    sh:maxCount 1;
  ], [
    sh:datatype xsd:integer;
    sh:path js:epicnessLevel;
    sh:name "epicness";
    sh:minCount 1;
    sh:maxCount 1;
  ].
```
and the following config file

```turtle
<myConfig> a js:MyEpicConfig;
  js:isEpic true;
  js:epicnessLevel 420.
```
results in the following json object inside the processor function.

```json
{
  "identifier": {"termType": "namedNode", "value": "myConfig"},
  "strategy": {
    "isEpic": true,
    "epicness": 420
  }
}
```

Note on cardinality: 
 - `sh:minCount` tells `rdf-lens` which properties are required and which properties are not, if `sh:minCount` is `0`, that parameter might be `null` inside the processor.
 - `sh:maxCount` tells `rdf-lens` which properties are singular or plural. If `sh:maxCount` is undefined or bigger than `1`, the parameter will be an **array**  inside the processor.

I hope that this example shows that processor shapes can be as easy as 4 different parameters each resulting in a string, or as difficult as creating nested objects that are not even fully defined in the processor definition file.

You can find more examples and already defined classes in the [rdf-lens repository](https://github.com/ajuvercr/rdf-lens).





