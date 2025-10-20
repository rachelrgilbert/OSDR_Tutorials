# Use the Biological Data API

[The OSDR Biological Data API](https://visualization.osdr.nasa.gov/biodata/api/) (BDAPI) allows users to
programmatically query and retrieve data and metadata from datasets contained in the
[NASA Open Science Data Repository](https://osdr.nasa.gov/bio/repo/) (NASA OSDR).

This tutorial uses the task of obtaining metadata and data for differential gene expression meta-analysis as an example
of its usage, providing minimal required information to achieve this, and building towards the complete API requests
from the ground up.

(For a more exhaustive, but inevitably more terse, description of the API's capabilities, please refer to the
[API's landing page](https://visualization.osdr.nasa.gov/biodata/api/))


## The ISA model

[NASA OSDR](https://osdr.nasa.gov/bio/repo/) datasets have their metadata encoded using the ISA model (ISA-Tab).
It is a standardized hierarchical format; the full specification is available on the
[ISA model](https://isa-specs.readthedocs.io/en/latest/) website, but the main piece of knowledge required for this
tutorial is that the metadata is stored under several levels of organization:

- *Investigation*-specific information, *e.g.* `investigation` &rarr; `study` &rarr; `study title`.
- *Assay*-specific information, *e.g.* `assay` &rarr; `parameter value` &rarr; `sequencing instrument`.
- *Study*-specific information:
    - *Factor values*, target metadata for the experiment: *i.e.* independent variables associated with samples that
      are purposely manipulated for the analysis.
      For example, when studying the effects of spaceflight, researchers will investigate spaceflight and ground
      control samples, and these metadata will be stored under `study` &rarr; `factor value` &rarr; `spaceflight`.
    - *Characteristics*, metadata that describes the samples but not manipulated by the experimentalist, *e.g.*,
      the organism in question (`study` &rarr; `characteristics` &rarr; `organism`).
    - *Parameter values*, all other ancillary metadata.


## Hierarchical metadata in the Biological Data API

The Biological Data API allows users to query for these metadata according to this model.
The full documentation is available on the [API's landing page](https://visualization.osdr.nasa.gov/biodata/api/), but
for the purposes of this tutorial, the main thing to know is that the ISA-Tab hierarchy described above can be
referenced directly; for example, the path `study` &rarr; `characteristics` &rarr; `organism` becomes a query
component in the form of `study.characteristics.organism`:

- By supplying this as a key to the API's metadata query endpoint (`/v2/query/metadata/`), we get a table of all
  organism annotations for all samples in the database, in the CSV format:
  [https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism)
- We can speficy *which* organism we're interested in by adding an equals sign and the value of interest, *e.g.*
  `study.characteristics.organism=mus musculus`.
  The API is case-insensitive by default (so it's OK to spell it in lower case), but keep in mind that spaces need to be
  encoded as `%20` to produce a valid URL (*i.e.*, `mus%20musculus`):
  [https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus)
- Inequalities are also allowed, here's everything that's *not* Mus musculus:
  [https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism!=mus%20musculus](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism!=mus%20musculus)


## Additional fields

Accession numbers (*e.g.* "OSD-4"), assay names (*e.g.* "OSD-4_transcription-profiling_dna-microarray_affymetrix") and
sample names (*e.g.* "Mmus_C57-6T_TMS_FLT_Rep4") aren't directly encoded in the ISA-Tab hierarchy, but BDAPI provides
them for convenience as `id.accession`, `id.assay name`, and `id.sample name` respectively:

    - *E.g.* `id.accession=OSD-37`

NASA OSDR also associates *files* with datasets and samples.
Very often, we'll need to find a file of a certain data type or with a certain name; for this purpose, BDAPI exposes
this information for convenience as `file.data type` (or its alias `file.datatype`), `file.file name` (`file.filename`):

    - E.g.* `file.datatype=pca`


## Query endpoints vs. REST endpoints

If you are ever not sure which metadata is available for which dataset, you can drop the last part of the key
(for example, instead of `study.characteristics.organism` you can just supply `study.characteristics`). Combining this
with *e.g.* `id.accession`, we obtain a table of all study characteristics for a given accession:

- [https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?id.accession=OSD-48&study.characteristics](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?id.accession=OSD-48&study.characteristics)

For the majority of analytical work, the metadata query endpoint (`/v2/query/metadata/`) and its sister *data* query
endpoint (`/v2/query/data/`) are to be used.

With that said, BDAPI also provides a REST interface, which can be used to *explore* the metadata.
The syntax for this endpoint is different, following the REST paradigm, and therefore more limited and not conducive to
complex query operations, but can be quite useful for *traversal* of the metadata.

You can read up on it on the [API's landing page](https://visualization.osdr.nasa.gov/biodata/api/), but we suggest to
use it only to "peek" at the metadata (for example, here's the metadata for all assays and all samples of dataset
OSD-48: [https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-48/assay/*/sample/*/](https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-48/assay/*/sample/*/)), and stick to the query interface for actual
analytical work.


## Obtaining metadata tables

Let's say we're interested in all `Mus musculus` datasets with `left kidney` samples that have been investigated for the
effects of `spaceflight`, and additionally have files of type `unnormalized counts` (*i.e.*, the files produced in the
course of an RNA-Seq differential expression analysis pipeline).

We can build a metadata request as follows:

    - For the organism: `study.characteristics.organism=mus%20musculus`
        - (recall that spaces must be converted to `%20`);
    - For the tissue "left kidney": `study.characteristics.material%20type=left%20kidney`
        - (tissues are always under the "material type" characteristic field; if you're not sure where annotations like
          "left kidney" -- or others -- reside, you can find this out for the future using a REST endpoint for a dataset:
          *e.g.*, visit [https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-25/assay/*/sample/*/](https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-25/assay/*/sample/*/) and search for "left kidney")
    - For spaceflight: `study.factor%20value.spaceflight`
        - (again, in this case, we've already mentioned that "spaceflight" is usually a factor value, so it's not
          surprising that it's found under this key; in general, you can similarly inspect REST endpoints to find out
          where such annotations are expected to be found)
        - Note that we did not specify what `spaceflight` is equal *to*; we just said `study.factor%20value.spaceflight`
          without an equals sign. This will return *all* values for all samples. We *could* have constrained it to a
          value, similarly to how we did it for the organism and the tissue type, though.
    - For the file datatype of "unnormalized counts": `file.datatype=unnormalized%20counts`

Finally, we append all these components to the metadata query endpoint (`/v2/query/metadata/`) of the API (so, the full
URL will start with `https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/`).

For URLs like these (not only in BDAPI, but universally), components are joined with an ampersand (`&`) and added to
the endpoint after the question mark (`?`); in other words, `https://some.url/?key=value&key=value&key=value`):

[https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts)

This URL will output a table of all NASA OSDR per-sample metadata and file information that matches the specified
constraints.


## Obtaining data tables

Tabular data from underlying files can be also obtained from BDAPI.
Such files include, among others, datatypes "unnormalized counts," "normalized counts," "differential expression,"
"pca."

**All you need to do to go from the metadata view to the data view is to change the endpoint from**
`/v2/query/metadata/` **to** `/v2/query/data`:

[https://visualization.osdr.nasa.gov/biodata/api/v2/query/data/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts](https://visualization.osdr.nasa.gov/biodata/api/v2/query/data/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts)

However, some requirements need to be met in order for BDAPI to produce such a table:

- Either the request resolves to a single underlying file (for instance, you've specified `id.accession`
  and *e.g.* `file.datatype=pca`: notice how the same file name figures in all rows of the
  [metadata table](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?id.accession=OSD-37&file.datatype=pca),
  which allows the API to produce a matching [data table](https://visualization.osdr.nasa.gov/biodata/api/v2/query/data/?id.accession=OSD-37&file.datatype=pca))
- Or the request resolves to multiple files that can be reasonably merged together; in our example of unnormalized
  counts for left kidney, this is exactly the case: unnormalized counts tables can be merged across multiple datasets,
  and batch effect correction *etc.* will need to be addressed by the user.
  Currently, "unnormalized counts" is the only datatype in BDAPI that can be merged this way and allow to produce tables
  like [the one we generated above](https://visualization.osdr.nasa.gov/biodata/api/v2/query/data/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts).


## Output formats

Up until this point, we have been obtaining the results from query endpoints in the CSV format (a sensible default for
tabular data), and from the REST endpoints in the JSON format (which is standard for REST).

BDAPI supports a number of output formats (see the formats section on the
[landing page](https://visualization.osdr.nasa.gov/biodata/api/#output-formats)).
The majority are machine-readable and usable for programmatic analyses (CSV, TSV, various flavors of JSON), but if your
goal is to *inspect* the data, you might be interested in `format=html`, *e.g.*:

- Tabular view: [https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts&format=html](https://visualization.osdr.nasa.gov/biodata/api/v2/query/metadata/?study.characteristics.organism=mus%20musculus&study.characteristics.material%20type=left%20kidney&study.factor%20value.spaceflight&file.datatype=unnormalized%20counts&format=html)
- JSON view: [https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-48/assay/*/sample/*/?format=html](https://visualization.osdr.nasa.gov/biodata/api/v2/dataset/OSD-48/assay/*/sample/*/?format=html)
