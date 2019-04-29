# Exercises

## Exercise 1

### Analyze and convert `neukoelln.csv` to different formats like JSON, XLSX & YAML:

```
# analyze the CSV file
$ head neukoelln.csv
# convert CSV to different formats
$ catmandu convert CSV to JSON < neukoelln.csv
$ catmandu convert CSV to XLSX --file neukoelln.xlsx < neukoelln.csv
$ catmandu convert CSV to YAML < neukoelln.csv
```

### Extract the column `geschlecht` from `neukoelln.xlsx` and count the different genders:

```
$ catmandu convert XLSX to CSV --fields geschlecht < neukoelln.xlsx | sort | uniq -c
```

### Extract the legal representatives `legalResp` from the EU transparency register `eu-transparency-register.xml` and flatten the data structure:

```
# analyze the XML file
$ less eu-transparency-register.xml
# extract <legalResp> elements 
$ catmandu convert XML --path '//legalResp' to YAML < eu-transparency-register.xml
# flatten the data structure and report number of items
$ catmandu convert -v XML --path '//legalResp' to TSV --fix 'collapse()' < eu-transparency-register.xml
```

How many representatives are registered?

### Create a (sub-)field statistic for the MARC collection `perl_books.mrc` and save as XLSX file:

```
$ catmandu convert MARC to Breaker --handler marc < perl_books.mrc > perl_books.breaker
$ catmandu breaker perl_books.breaker
$ catmandu breaker --as TSV perl_books.breaker > perl_books_stats.tsv
$ less perl_books_stats.tsv
```

Name the most common fields.

### Convert the MARC collection `perl_books.mrc` to the human-readable serialization MARCMaker:

```
$ catmandu convert MARC to MARC --type MARCMaker < perl_books.mrc > perl_books.mrk
$ less perl_books.mrk
``` 

Describe the data format.

### Query the ZDB SRU interface (http://services.dnb.de/sru/zdb?operation=explain&version=1.1) for the ISSN '1940-5758':

```
$ catmandu convert SRU --base  http://services.dnb.de/sru/zdb --recordSchema oai_dc --parser simple --query 'iss=1940-5758' to YAML
```

What's the name of the journal?

### Get all libraries of Berlin via the lobid-organisation (http://lobid.org) JSON API:

```
$ catmandu convert getJSON --from 'http://lobid.org/organisations/search?q=location.address.addressLocality%3ABerlin&format=json' to YAML
```

Use the `--proxy 'http://proxy.sbb.spk-berlin.de:3128'` option on the `getJSON` command, if you have trouble to reach lobid.org.

How many libraries are located in Berlin?

## Exercise 2

### Analyze the dataset `libraries.json` and extract all library names and ISIL using the JSONPath dot notation:

```
# analyze the JSON file
$ less libraries.json
# extract all names and ISIL
$ catmandu convert JSON to CSV --fields name,isil --fix 'copy_field(properties.name,name);copy_field("properties.ref:isil",isil)' < libraries.json
# get only libraries with an ISIL
$ catmandu convert JSON to CSV --fields name,isil --fix 'copy_field(properties.name,name);copy_field("properties.ref:isil",isil);select exists(isil)' < libraries.json
```

How many of the libraries have an ISIL?

### Analyze internal data structure of MARC records in Catmandu. Extract the tag of the 6th field and the value of the 1th subfield. 

```
# analyze the MARC file
$ catmandu convert MARC to YAML < perl_books.mrc
# extract tag and first subfield value from the 6th field
$ catmandu convert MARC to CSV --fields tag,subf --fix 'copy_field(record.6.0,tag);copy_field(record.6.4,subf)' < perl_books.mrc
``` 

### Extract all titles (MARC 245, non repeatable) from the collection `perl_books.mrc`: 

```
$ catmandu convert MARC to CSV --fields title --fix 'marc_map(245a,title)' < perl_books.mrc
``` 

### Extract all personal names (MARC 700, repeatable) from the collection `perl_books.mrc` and count them: 

```
$ catmandu convert MARC to YAML --fix 'marc_map(700a,dc_contributor,split:1);remove_field(record)' < perl_books.mrc
$ catmandu convert MARC to CSV --fields dc_contributor --fix 'marc_map(700a,dc_contributor,split:1);count(dc_contributor)' < perl_books.mrc
$ catmandu convert MARC to CSV --header 0 --fields dc_contributor --fix 'marc_map(700a,dc_contributor,split:1);count(dc_contributor)' < perl_books.mrc | sort | uniq -c
``` 

How many books have more than 3 contributors?

### Extract the language codes from the MARC collection `perl_books.mrc` and look them up in the dictionary `dict_languages.csv`: 

```
$ catmandu convert MARC to YAML --fix 'marc_map(008/35-37,dc_language);lookup(dc_language,dict_languages.csv);remove_field(record)' < perl_books.mrc
$ catmandu convert MARC to CSV --fields dc_language --header 0 --fix 'marc_map(008/35-37,dc_language);lookup(dc_language,dict_languages.csv)' < perl_books.mrc | sort | uniq -c
```

How many books have no language code?

### Select records which conform the default MARC schema:

```
$ catmandu convert -v MARC to MARC --type MARCMaker --fix 'select valid(.,MARC)' < perl_books.mrc
```

How many records validate against the default schema?

### Export non valid records and their error messages:

```
$ less validate.fix
$ catmandu convert -v MARC to Null --var filename=non_valid \
--fix validate.fix < perl_books.mrc
$ less non_valid.jsonl
$ less non_valid.mrk
```

What is the most common error?

## Exercise 3

### Convert the MARC collection `perl_books.mrc` to Dublin Core with the fix `marc2dc.fix`:

```
$ head -n 100 marc2dc.fix
$ catmandu convert MARC to YAML --fix marc2dc.fix --pretty 1 < perl_books.mrc
```

### Convert the MARC collection `perl_books.mrc` to Dublin Core with the fix `marc2dc.fix` and import it to MongoDB

```
$ catmandu import -v MARC to MongoDB --database_name perl_books --fix marc2dc.fix < perl_books.mrc
$ mongo
> use perl_books
> db.data.find()
```

### Export all books published before 2000.

```
$ catmandu export -v MongoDB --database_name perl_books --query '{"dc_date": {"$lt":"2000"}}' to YAML < perl_books.mrc
```

### Export all books written by 'Wall, Larry'.

```
$ catmandu export -v MongoDB --database_name perl_books --query '{"dc_contributor":"Wall, Larry"}' to YAML < perl_books.mrc
```

### Create a CQL mapping for the data in the MongoDB:

```
nano catmandu.yml
---
store:
  cql:
    package: MongoDB
    options:
      database_name: perl_books
      bags:
        data:
          cql_mapping:
            default_index: all
            indexes:
              author:
                op:
                  'any': true
                  'all': true
                  '=': true
                  'exact': true
                field: 'dc_contributor'
              subject:
                op:
                  'any': true
                  'all': true
                  '=': true
                  'exact': true
                field: 'dc_subject'
              year:
                op:
                  'any': true
                  'all': true
                  '=': true
                  '>': true
                  '>=': true
                  '<': true
                  '<=': true
                  'exact': true
                field: 'dc_date'
...
```

### Make an CQL query for all books written by `Christiansen`:

```
$ catmandu export -v cql --cql-query 'author all Christiansen' to YAML
``` 

### Make an CQL query for all books published after `2000`

```
$ catmandu export -v cql --cql-query 'year > 2000' to YAML
``` 

### Make an CQL query for all books belongs to DDC class `005`

```
$ catmandu export -v cql --cql-query 'subject any 005' to YAML
``` 

## Excercise 4

### Create a new Fix script `lookup_viaf.fix` to match `dc_contributor` names against the VIAF API.

```
$ nano lookup_viaf.fix
# VIAF API https://platform.worldcat.org/api-explorer/apis/VIAF

# only if contributor exists
if exists(dc_contributor)

    # iterate over all contributor values
    # varibale 'current' is the current contributor
    do list(path:dc_contributor, var:current)

        # a lookup.query variable saves the current name
        copy_field(current,lookup.query)

        # let Catmandu make the request to JSON API and store results in lookup.viaf variable
        get_json("http://viaf.org/viaf/AutoSuggest?query={query}", path: lookup.viaf, vars: lookup)

        # on success lookup.viaf.result should be a list 
        if exists(lookup.viaf.result.0)

            # check if we got a satisfying score
            if greater_than(lookup.viaf.result.0.score,1000)

                # prepend base URL to VIAF ID
                prepend(lookup.viaf.result.0.viafid,'https://viaf.org/viaf/')

                # append VIAF URI to a new dct_contributor field
                copy_field(lookup.viaf.result.0.viafid,dct_contributor.$append)
            end       
        end

        # delete temporary lookup variable
        remove_field(lookup)
    end
end
```

### Run this fix in combination with the `marc2dc.fix` and check the new element `dct_contributor`

```
$ catmandu convert MARC to YAML --fix marc2dc.fix --fix lookup_viaf.fix < perl_books.mrc
```
