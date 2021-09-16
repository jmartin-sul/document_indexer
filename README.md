# indexing document text and metadata for analysis

using a bunch of open source tools to examine a whole bunch of documents
* Apache Tika (document text and metadata processing)
* Solr (text indexing, search, and faceting)
* ICIJ Extract (for processing documents into e.g. a Solr index)
* Blacklight (Ruby on Rails engine for exploring data stored in a Solr index)

## setup the processing tools

* `docker compose up` with the default `docker-compose.yml` (will create `var_solr` if it doesn't exist)
* `docker exec -it document_indexer_solr_1 solr create_core -c document_core` from host CLI
  * see https://github.com/docker-solr/docker-solr/#manually
* add below `lib` and `requestHandler` stanzas to the appropriate spots in the `solrconfig.xml` that core creation generated
* _not sure this is necessary?_ save the example linked here as `schema.xml` (sibling of `solrconfig.xml` in `var_solr/data/document_core/conf`, there shouldn't be one there yet to overwrite)
  * https://github.com/ICIJ/extract/wiki/Metadata#example-solr-schema
* install apache tika (can be done via homebrew on mac)
  * https://tika.apache.org/2.0.0/gettingstarted.html
  * alternatively: https://github.com/apache/tika-docker _(could incorporate into docker-compose.yml)_
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


## index some documents

### test on one PDF using `curl`

```sh
$ `curl 'http://localhost:8983/solr/document_core/update/extract?literal.id=helloworld&commit=true' -F "myfile=@test.pdf"`
{
  "responseHeader":{
    "status":0,
    "QTime":2179}}
```

### in bulk using extract

run something like:
```sh
# from the extract project directory after building
$ java -jar extract-cli/target/extract-cli-3.8.1.jar spew --ocr no -o solr -s 'http://localhost:8983/solr/document_core' --commitInterval 500  'my_cool_pdf_directory'
```

### TODO - try calling tika as a CLI process from a script, no extract

extract might be overkill for me.  try calling tika on the PDFs (since those are the documents you have at the moment) and then indexing fields of my choice from parsing, using python or ruby.  i had trouble figuring out how to get tika to index the fields i wanted.

both pysolr and rsolr seem fine, tika bindings for python seem better than those for ruby, so prob easier to write indexing code in python than ruby?  would be easy to do simple text extraction using tika and create a document based on that.  could generate same document IDs as ICIJ extract (i think just a hash of the document content?).

https://github.com/ICIJ/extract/blob/master/extract-lib/src/main/java/org/icij/extract/document/DocumentFactory.java#L44

https://solr.apache.org/guide/8_8/overview-of-documents-fields-and-schema-design.html
https://github.com/projectblacklight/blacklight/wiki/Blacklight-configuration

https://github.com/chrismattmann/tika-python
https://github.com/django-haystack/pysolr/
https://github.com/mrcsparker/ruby_tika_app
https://github.com/duke-libraries/tika-client
https://github.com/kanety/tikarb
https://github.com/rsolr/rsolr

this seems to do something similar to what i'm trying to do
https://github.com/EricLondon/Docker-Rails-Tika-Elasticsearch / https://ericlondon.com/2017/02/01/integrate-tika-rest-service-with-rails-paperclip-attachments-to-extract-text-from-pdf-documents-and-store-in-elasticsearch.html

not sure how useful this is, but maybe? https://github.com/chrismattmann/tika-similarity

## explore your indexed documents

### done already in this repo, just in case it helps spin up similar in the future...

#### install rails

https://guides.rubyonrails.org/getting_started.html

`rails new . --skip-spring --skip-listen`

#### install blacklight

https://github.com/projectblacklight/blacklight/wiki/Quickstart
* confirm the dependencies (ruby, solr, node, yarn, etc) are installed
* go to the "Creating a new application the hard way" section -- https://github.com/projectblacklight/blacklight/wiki/Quickstart#creating-a-new-application-the-hard-way
* skip solr_wrapper because you're already running solr in docker
* `rails generate blacklight:install --devise --solr_version=latest`
* `bin/rails db:migrate`
* `rails s`
* modify the default catalog controller a bunch because it expects a totally different schema than you get from tika throwing PDFs at solr
