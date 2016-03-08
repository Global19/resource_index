# NCBO Resource Index

[![Build Status](https://travis-ci.org/ncbo/resource_index.svg?branch=master)](https://travis-ci.org/ncbo/resource_index)

The NCBO Resource Index project process a variety of biomedical resources (IE collections of documents or data)
and generates annotations using classes from BioPortal.

In addition to the annotations, the Resource Index project can generate a co-occurence matrix for labels and
classes found in BioPortal.

## Dependencies
- Ruby 2.x
- Elasticsearch 1.x
- mgrep (available on the [NCBO Virtual Appliance](http://www.bioontology.org/wiki/index.php/Category:NCBO_Virtual_Appliance))
- MySQL (used for defining indidivual resources, their metadata, and a normalized version of their documents/data)

## Installation

    git clone https://github.com/ncbo/resource_index.git
    cd resource_index
    bundle

## Usage

The Resource Index is comprised of two parts: 1) Population and 2) Deployment.

For populating, a ResourceIndex::Population::Manager object can be instantiated by passing in a resource object along with configuration options. Configuration options and their defaults are detailed in `lib/ncbo_resource_index/population/population.rb`.

Here is a sample, basic script that will configure and run a population job:

    require 'resource_index'

    # this mysql db should contain resources and their data
    RI.config(username: "user", password: "pass", host: "localhost")
    res = RI::Resource.find("PM")
    populator = RI::Population::Manager.new(res,
    { annotator_redis_host: "redis1",
      mgrep_host: "mgrep1",
      goo_host: "4store1",
      goo_port: 8080,
      population_threads: 4,
      mail_recipients: "user@example.org" })
    populator.populate(delete_old: true)

During the population process, the emails listed in `mail_recipients` will get an email if the process encounters an error or finishes.

# RI Design

## Workflow Overview

Resources come from a variety of sources that are processed through ETL using a Resource Access Tool

## General Workflow

### From Scratch

- Introspect resource layout (column data) and create corresponding ElasticSearch document mapping
- Annotate documents using mgrep and annotator cache
- Store documents and annotations in Elasticsearch
    - Annotations are stored as follows:
        + Direct (set of int)
        + Ancestors (set of int)
    - Class ids are hashes of the ontology acronym and class URI created with xxhash

### Updating

Currently, updates are not supported.

In the future, it might be possible to update the Resource Index by looking at the dictionary generated by annotator and using the new labels to run a search in ElasticSearch and then just update the documents that contain hits for the new labels. The theory is that the dictionary actually doesn't change that much and so just looking for hits on the new labels would be better than re-annotating the entire corpus.

## Elasticsearch Configuration

The following is an example mapping for a PubMed abstract. The annotations are nested and just the tokens get stored, not the original json. Nesting them puts them in a separate index, allowing you to search just the annotations very quickly.

```json
{
    "mappings": {
        "citation": {
            "_source" : {
              "includes": ["title", "abstract"],
              "excludes": ["annotations"]
            },
            "properties": {
                "title": {
                  "type": "string"
                },
                "abstract": {
                  "type": "string"
                },
                "annotations": {
                    "type": "nested",
                    "properties": {
                        "direct": {"type": "long", "store": false, "include_in_all": false},
                        "ancestors": {"type": "long", "store": false, "include_in_all": false},
                    }
                }
            }
        }
    }
}
```

## Ruby Performance

The default MRI implementation of Ruby is not great at efficiently handling threads. There is a version of Ruby that can run in the JVM called Jruby, and switching to this provides a nice performance boost, especially as we can run the population process in threads to take advantage of a shared in-memory cache.

The code is written to work in both MRI or Jruby, but population should always be done using Jruby.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
