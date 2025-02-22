# Mock Data

To supplement real world data, both public and private, I have added a
mock data generator to the project. This generator is used to create
correctly formatted data for testing purposes.

Data created using [the data mocking model](../reanalysis/data_model.py)
is generated in either Hail Table or MatrixTable format, and contains
all basic fields required to satisfy the AIP data contract. This
conforms to the format which would be generated using the [CPG pipeline](https://github.com/populationgenomics/production-pipelines/blob/main/cpg_workflows/query_modules/seqr_loader.py#L18).

The fields generated by the model are only a subset of the overall
CPG pipeline fields, but they cover all the fields required by this
codebase, and satisfy the [AIP data contract](../reanalysis/hail_audit.py).

Although data generated by the model is returned in a Hail native format,
either of the available formats can be translated to a correctly formatted
VCF file using the [Hail export_vcf](https://hail.is/docs/0.2/methods/impex.html#hail.methods.export_vcf) method.

## Purpose

Although we have access to a wide range of diagnostic and research cohorts,
to test specific logical corner cases or to iterate design of new elements
it is useful to be able to create data which has certain fixed attributes.
Although there are existing ways to do this at various scales, e.g. manually
modifying a real VCF file, or utilising tools such as [GeneBreaker](https://pubmed.ncbi.nlm.nih.gov/33368787/)
to generate partially synthetic data, these methods fail in a few key ways:

1. They are low throughput, and require substantial manual effort to create
   a large number of variants/samples.
2. They aim to be biologically valid, usually to test a broad range of
   scenarios. This means that they are not necessarily optimised for
   testing specific logical corner cases.
3. The data is not in an AIP-ready format, which creates issues around
   requiring re-annotation by a CPG-like process to be ready to use ($$$).

Instead of working from the ground up (either starting from reads or a VCF
where a real variant with known attributes is spiked in), I'm starting at
the end of the data generation process. Instead of going to the effort of
creating variants of a specific type, we are choosing the exact annotations
we want a variant to have post-annotation, and creating them.

## Usage

The mock data generator is simple - a number of models are defined in [data_model.py](../reanalysis/data_model.py),
each corresponding to a discrete section of the VEP/gnomAD annotated data:

* Base fields (chromosome, position, reference, alternate, etc.)
* AF (exomes/genomes)
* SpliceAI results
* CADD results
* ClinVar annotation
* DBnsfp annotation
* Transcript Consequences
* Entry Fields (sample Genotype, read depths, qualities)

Almost all of these model fields contain neutral defaults, so the only
fields required to create a variant object are:

* Coordinates (Contig and Position)
* Reference and Alternate alleles
* Transcript Consequence(s), each with a Gene ID and Symbol

Example usage with relevant comments can be found in [this script](../reanalysis/model_go_brrr.py).

When a variant is created, it will be JSON serialised and written into a
temporary file. This file is then read into a Hail Table, decoded using
a specific schema (getting around some weird Hail issues with Pandas parsing),
written to a file, then returned - you can write it to a new location, use it
for testing, or export it into a new format (e.g. VCF).

Optionally, sample-level data can also be added. The data model contains an
`Entry` field, which is a fairly complex object containing many sample-level
fields, and the corresponding default schema. If non-`Entry` sample data is
used, you also need to provide an appropriate JSON schema to decode it. This
can be used to supply genotypes, read depths, qualities, etc., as well as
giving a specific phase set to one or more variants, which will lead to a
phased interpretation of the genotypes downstream.

## Scenarios

### Unit Testing

Currently unit tests make use of data mocking on top of real data

* Ingest a VCF as a MT, distributed as a test fixture
* For each test, overwrite the annotations with specific values
* Tests that methods interpreting those values function as expected

The data mocking functionality allows for replacement of this with new
behaviour:

* Create a MT with a single variant and specific annotations
* Run the test on this MT, check for correct results

### Category Design

Imagine we want to create a new category which leverages the `population
allele frequencies`, `variant consequences`, and `spliceAI scores`. We can
create completely synthetic variants that have the required properties,
as well as variations on all properties to target all edge cases.

### MOI tests

If we want to test the inheritance models, we can create one or more
variants, then add specific sample-level attributes to create any test
scenario:

* We can use any family structure - singleton, trio, duo, quad, 5-gen
  family, etc.
* We can use any genotypes - all HomRef, all HomAlt, all Het, some
  missing, etc.
* We can use any phase sets - all phased, all unphased, some phased,
  some unphased, etc.

Coupled with that, we can provide any artificial pedigrees, making all
family members affected, unaffected, or any variation thereof.

We can also provide any PanelApp data, which will lead to the
interpretation of the variant(s) using a Dominant, Recessive, X-Linked,
or any other included MOI.

## Caveats

By using this data mocking model, we are relying on AIP's ability to use
the data it is provided, without checking its biological validity.

If a variant in the AIP input is marked as being a Frameshift, Pathogenic
in ClinVar, and with a high SpliceAI & CADD score, this becomes its own
source of truth. Within AIP the variant will be processed based on its
annotations, and we do not check that the annotations are correct. We
do not even check if the supplied reference base(s) match the reference
genome.

If this data model is used to create synthetic data, the data format may
be correct, but there is no guarantee that the annotations will be useful.
Users should be careful to create variants which are biologically valid,
e.g. using input from a source of real data such as ClinVar, or gnomAD.

In those contexts, this functionality may still provide some utility:

* Provide real variants in an easily parsed format
* Generate variant objects for each, adding required minimum annotations
* For each variant, generate as many samples as you want,
    each with a specific genotype, phase set, etc.
* Export the partially-synthetic data to VCF, and use as input to any
    analysis process
