# indexing PDF text and metadata for analysis

using a bunch of open source tools to examine a whole bunch of documents
* Apache Tika (document text and metadata processing)
* Solr (text indexing, search, and faceting)
* ICIJ Extract (for processing documents into e.g. a Solr index)
* Blacklight (Ruby on Rails engine for exploring data stored in a Solr index)

## setup the processing tools

* `docker compose up` with the default `docker-compose.yml` (will create `var_solr` if it doesn't exist)
* `docker exec -it pdf_indexer_solr_1 solr create_core -c pdf_core` from host CLI
  * see https://github.com/docker-solr/docker-solr/#manually
* add below `lib` and `requestHandler` stanzas to the appropriate spots in the `solrconfig.xml` that core creation generated
* _not sure this is necessary?_ save the example linked here as `schema.xml` (sibling of `solrconfig.xml` in `var_solr/data/pdf_core/conf`, there shouldn't be one there yet to overwrite)
  * https://github.com/ICIJ/extract/wiki/Metadata#example-solr-schema
* install apache tika (can be done via homebrew on mac)
  * https://tika.apache.org/2.0.0/gettingstarted.html
* install ICIJ Extract
  * https://github.com/ICIJ/extract/ -- "A cross-platform command line tool for parallelized, distributed content-extraction. Built on top of Apache Tika and an essential part of the engineering behind the Panama Papers, Swiss Leaks and Luxembourg Leaks investigations."
  * see command below
    * requires installing maven (also available via homebrew, for mac)
    * consider installing jEnv (similar to rbenv and pyenv) if you have multiple java versions to manage -- https://github.com/jenv/jenv

### additional `solrconfig.xml`

```xml
  <lib dir="${solr.install.dir:../../..}/contrib/extraction/lib" regex=".*\.jar" />
  <lib dir="${solr.install.dir:../../..}/dist/" regex="solr-cell-\d.*\.jar" />

  <requestHandler name="/update/extract"
                startup="lazy"
                class="solr.extraction.ExtractingRequestHandler" >
    <lst name="defaults">
      <str name="lowernames">true</str>
      <str name="fmap.content">_text_</str>
    </lst>
  </requestHandler>
```

### build the jar for ICIJ's extract tool

```sh
# to successfully build extract.jar, i got errors on the javadoc task and the gpg task, and signing is only necessary for publishing the build, not running it locally
$ mvn install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Dgpg.skip=true
```


## index some PDFs

### test on one PDF using `curl`

```sh
$ `curl 'http://localhost:8983/solr/pdf_core/update/extract?literal.id=helloworld&commit=true' -F "myfile=@test.pdf"`
{
  "responseHeader":{
    "status":0,
    "QTime":2179}}
```

### in bulk using extract

run something like:
```sh
# from the extract project directory after building
$ java -jar extract-cli/target/extract-cli-3.8.1.jar spew --ocr no -o solr -s 'http://localhost:8983/solr/pdf_core' --commitInterval 500  'my_cool_pdf_directory'
```

## explore your indexed PDFs


