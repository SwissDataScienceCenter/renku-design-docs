- Start Date: 22-07-2022
- Status: Proposed

# New metadata fields for datasets

## Summary

Add standard metadata fields to make Renku datasets more FAIR.

## Motivation

As a Renku user, I want to easily annotate my datasets with standard metadata. Renku datasets support custom annotations as JSON-ld files. Although these annotations are available in the project graph, they are not in the global KG. Furthermore, the process of adding custom metadata is tedious and error prone (requires writing JSON-ld by hand). Currently, some crucial default metadata fields (e.g. license) are missing from datasets.

Datasets metadata should support common fields required for FAIR-ification.

## Design Detail

### User research data

We can leverage information from 3 sources to identify potential fields. In order of decreasing priority: FAIR recommendations, User feedback, User persona.

**FAIR recommendations**

* F1: (Meta)data are assigned a globally unique and persistent identifier
  + _Ideally we would use something like ROR for **affiliation**_
* R1.1: data are released with a clear and accessible data usage **license**

**Feedback**

* I would like to know the total dataset **size** to adjust session ressources and avoid running out of disk space
* Dataset imports are slow
  + _A **size** attribute may help by showing an ETA or even “large dataset” warning in the future._
* I would like to search entities by **affiliation** and **topic** in the search page

**Persona**

* No visibility on my project
  + _Unique **affiliation** (ROR, ORCID) + **topic** may help with that_

### Proposed fields

The following fields can be identified from the data above. They are listed in order of priority with the following logic: FAIR recommendation > user feedback > user persona.

**License**

Data usage license. The license cannot live _only_ as metadata and there must also be a file in the project. Therefore, this field should be populated when the file is added. In the future, Renku could propose a (graphical/no code) way to choose a license, which would both populate the metadata field and add the file. Here, we simply propose to make this optional field available for future use.

* **Reason:** FAIR compliance
* **Mapping:** [license](https://schema.org/license)
* **Value:** [URL](https://schema.org/URL) from https://spdx.org/licenses
* **Example:**

```json
"https://schema.org/license" : "https://spdx.org/licenses/MIT.html"
```
* **Alternative with DigitalDocument**:
```json
{
  "@context": "http://schema.org/",
  "@type": "DigitalDocument",
  "name": "MIT License",
  "identifier": "MIT",
  "url": "https://spdx.org/licenses/MIT.html",
  "author": "Alice",
  "text": "[license text]"
}
```

> Note: See [here](https://github.com/spdx/license-list-data/blob/master/accessingLicenses.md) for programmatic access to the list of licenses and the details of each license. If a license already exists in the project, SPDX also provides a [license matcher](https://github.com/spdx/spdx-license-matcher) to infer the license type from the text.

**Topic**

The research field. Can be a flat list of keywords, or ideally fine grained terms with hierarchical relationships. This would also be relevant for projects. Topics could be differentiated from arbitrary keywords by using DefinedTerm instead of Text. This would allow a simple implementation similar to [Dataverse](https://demo.dataverse.org/dataverse/demo/search), which uses a (short) flat list of terms. 

* **Reason:** User persona + user feedback
* **Mapping:** [about](https://schema.org/keywords)
* **Value:** [DefinedTerm](https://schema.org/DefinedTerm)
* **Example:**

```json
"http://schema.org/keywords": [
      {
        "@value": "demo"
      },
      {
        "@type": "DefinedTerm",
        "@id": "https://www.wikidata.org/wiki/Q2539",
        "name": "Machine learning",
      }
]
```

> Note: A more complex approach similar to [Swiss UBase](https://www.swissubase.ch/en/) with hierarchical fine grained topics may be desirable but harder to implement.["Data Science Education Ontology (DSEO)"](https://fairsharing.org/FAIRsharing.7p0xdg) has a [Domain](https://bioportal.bioontology.org/ontologies/DSEO/?p=classes&conceptid=http%3A%2F%2Fbigdatau.org%2Fdseo%23domain) class which subclasses owl#Thing. Wikidata's [academic disciplines](https://www.wikidata.org/wiki/Q336) also provide unique identifiers and with a herarchy encoded through `instance_of` / `has_parts` properties.

**Size**

* **Reason:** user feedback
* **Mapping:** [size](https://schema.org/size)
* **Value:** [Text](https://schema.org/Text)
* **Example:**

```json
"https://schema.org/size": {
    "@type": "QuantitativeValue",
    "value": 130,
    "unitCode": "AD"
}
```

> Note: It may not be possible to set it for external datasets. Also, [contentSize](https://schema.org/contentSize) could be more appropriate, but for now it only supports text.

## Drawbacks

> Why should we *not* do this? Please consider the impact on users,
on the integration of this change with other existing and planned features etc.

Proliferation of unnecessary fields may impact the performance and size of the knowledge graph, thus negatively affect user-experience.

> There are tradeoffs to choosing any path, please attempt to identify them here.

This may introduce problems with backwards compatibility, since the metadata structure would be modified.

## Rationale and Alternatives

> Why is this design the best in the space of possible designs?

Adding generic information directly in the dataset metadata will make it easier for users to provide crucial context on their data. It is a first step towards encouraging them to fill in these fields.

Currently, affiliation is a free-text value, which means that different values can refer to the same institution (e.g. UNIL and University of Lausanne), or a value may be ambiguous (UCD for University College Dublin or University of California, Davis).

> What other designs have been considered and what is the rationale for not choosing them?

Another option would be to keep these fields as custom annotations and provide tools to help users with data entry. The main issue with this solution is that the fields will not be available in the global KG and thus cannot be used in the search (e.g. to filter datasets in "Machine learning").

> What is the impact of not doing this?

Datasets are harder to find, less citable and usage guidelines are unspecified.

## Unresolved questions

> What parts of the design do you expect to resolve through the RFC process before this gets merged?

Should some of these fields be editable directly by users on the command line ? If so, how to prevent invalid values ?

Would some of these changes introduce incompatibility in (ex/im)porting datasets to/from third party providers ?

How do we deal with missing data ?

Is DSEO's `Domain` class a desirable solution for research topic ?

> What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

Setting the values (e.g. license, affiliation) in the UI, potentially through forms with autosuggestion.

Affiliation was deemed out of scope, as it implies modifying important changes to the user's metadata and not the Dataset's. The initial proposal is shown below for reference.

<details>
<summary>Affiliation field (out of scope)</summary>

**Affiliation**

For visibility and citability, affiliations should have a unique identifier. Affiliation already exists as a property of the dataset's creator. Ideally we could add a unique identifier to it. Rather than a property of the Dataset, this may be a property of the creator's `affiliation` and would also be available for projects.

* **Reason:** FAIR compliance + user persona
* **Mapping:** [creator](https://schema.org/creator) > [affiliation](https://schema.org/affiliation) > [Organization](https://schema.org/Organization) > [identifier](https://schema.org/identifier)
* **Value:** [URL](https://schema.org/URL) from https://ror.org/
* **Example:**
```json
"http://schema.org/affiliation": [
    {
    "@type": "Organization",
    "name": "EPFL",
    "identifier": "https://ror.org/02s376052"
    }]
```

> Note: ROR has a public REST API which could be used to autosuggest records in a text form in the future. More info [here](https://ror.readme.io/docs/create-affiliation-selection-dropdowntypeahead-widgets)
</details>
