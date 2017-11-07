# Grails Hibernate Search Plugin

This plugin aims to integrate Hibernate Search features to Grails in very few steps.

- [Grails Hibernate Search Plugin](#grails-hibernate-search-plugin)
  * [Getting started](#getting-started)
    + [Configuration](#configuration)
    + [Indexing](#indexing)
    + [Search](#search)
    + [Analysis](#analysis)
    + [Filters](#filters)
    + [Options](#options)
  * [Examples](#examples)
  * [Change log](#change-log)
  * [Authors](#authors)
  * [Development / Contribution](#development-contribution)
  * [License](#license)



## Getting started

If you don't want to start from the [template project](#examples), you could start a fresh project:

Add the following repository
```
  repositories { maven { url "http://dl.bintray.com/lgrignon/plugins" } }
```

And add the following to your dependencies
```
  compile("org.grails.plugins:hibernate-search:2.1.1")
  compile("org.grails.plugins:hibernate5:6.1.4")
  compile("org.grails.plugins:cache")
  compile("org.hibernate:hibernate-core:5.2.9.Final")
  compile("org.hibernate:hibernate-ehcache:5.2.9.Final")
```

### Configuration

By default, the plugin stores your indexes in this directory:

```
 ~/.grails/${grailsVersion}/projects/${yourProjectName}/lucene-index/development/
```

You can override this configuration in your application.yml

```yml
hibernate:
    cache:
        use_second_level_cache: true
        use_query_cache: true
        provider_class: net.sf.ehcache.hibernate.EhCacheProvider
        region:
            factory_class: org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory
    search:
        default:
        	indexBase: '/path/to/your/indexes'
            indexmanager: near-real-time
            directory_provider: filesystem
```

You can also define the path to your indexes with JNDI configuration as following:

```yml
hibernate:
    cache:
        use_second_level_cache: true
        use_query_cache: true
        provider_class: net.sf.ehcache.hibernate.EhCacheProvider
        region:
            factory_class: org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory
    search:
        default:
            indexBaseJndiName: 'java:comp/env/luceneIndexBase'
            directory_provider: filesystem
```

###  Indexing

#### Mark your domain classes as indexable

Add a static search closure as following:

**Note**: You can use properties from super class and traits with no additional configuration (since 2.0.2)

```groovy
class MyDomainClass {

    String author
    String body
    Date publishedDate
    String summary
    String title
    Status status
    Double price
    Integer someInteger

    enum Status {
        DISABLED, PENDING, ENABLED
    }

    static hasMany = [categories: Category, items: Item]

    static search = {
        // fields
        author index: 'yes'
        body termVector: 'with_positions'
        publishedDate date: 'day'
        summary boost: 5.9
        title index: 'yes'
        status index: 'yes'
        categories indexEmbedded: true
        items indexEmbedded: [depth: 2] // configure the depth indexing
        price numeric: 2, analyze: false
        someInteger index: 'yes', bridge: ['class': PaddedIntegerBridge, params: ['padding': 10]]

        // support for classBridge
        classBridge = ['class': MyClassBridge, params: [myParam: "4"]]
    }

}
```

This static property indicates which fields should be indexed and describes how the field has to be indexed.

Also, the plugin lets you to mark your domain classes as indexable with the Hibernate Search annotations.

```groovy
@Indexed
@ClassBridge(
     impl = MyClassBridge,
     params = @Parameter( name="myParam", value="4" ) )
class MyDomainClass {

    // when using annotations, id is required to define DocumentId
    @DocumentId
    Long id

    @Field(index=Index.YES)
    String author
    
    @Field(index=Index.YES)
    String body
    
    @Field
    @DateBridge(resolution=Resolution.DAY)
    Date publishedDate
    
    @Field(index=Index.YES)
    String summary
    
    @Field(index=Index.YES)
    String title
    
    @Field(index=Index.YES)
    Status status
    
    @Field 
    @NumericField( precisionStep = 2)
    Double price
    
    @Field(index=Index.YES)
    @FieldBridge(impl = PaddedIntegerBridge.class, params = @Parameter(name="padding", value="10"))
    Integer someInteger

    enum Status {
        DISABLED, PENDING, ENABLED
    }

    @IndexedEmbedded
    Set categories

    @IndexedEmbedded(depth = 2)
    Set items

    static hasMany = [categories: Category, items: Item]

}
```

#### Create index for existing data

The plugin lets you to create index of any indexed entity as following:

```groovy
MyDomainClass.search().createIndexAndWait()
```

This method relies on MassIndexer and can be configured like this:

```groovy

MyDomainClass.search().createIndexAndWait {
   ...
   batchSizeToLoadObjects 25
   cacheMode org.hibernate.CacheMode.NORMAL
   threadsToLoadObjects 5
   ...
}

```

#### Create index for existing data using script

Also, the plugin provides you a gant script to create index for data already present in your database:

##### Create Lucene index for all indexed entities

```shell
grails hs-create-index
```

##### Create Lucene index for a given domain class

```shell
grails hs-create-index com.test.MyDomainClass
```

#### Manual index changes

##### Adding instances to index

```groovy

// index only updated at commit time
MyDomainClass.search().withTransaction { transaction ->
   MyDomainClass.findAll().each {
      it.search().index()
   }
}
```

##### Deleting instances from index

```groovy

// index only updated at commit time
MyDomainClass.search().withTransaction { transaction ->
   
   MyDomainClass.get(3).search().purge()
   
}
```

To remove all entities of a given type, you could use the following purgeAll method:

```groovy

// index only updated at commit time
MyDomainClass.search().withTransaction {
   MyDomainClass.search().purgeAll()
}
```


##### Rebuild index on start

Hibernate Search offers an option to rebuild the whole index using the MassIndexer API. This plugin provides a configuration which lets you to rebuild automatically your indexes on startup.

To use the default options of the MassIndexer API, simply provide this option into your application.groovy:

```groovy


grails.plugins.hibernatesearch = {
    rebuildIndexOnStart true
}

```

If you need to tune the MassIndexer API, you could specify options with a closure as following:

```groovy

grails.plugins.hibernatesearch = {

    rebuildIndexOnStart {
		batchSizeToLoadObjects 30 
		threadsForSubsequentFetching 8 	
		threadsToLoadObjects 4 
		threadsForIndexWriter 3 
		cacheMode CacheMode.NORMAL
    }

}

```


### Search

The plugin provides you dynamic method to search for indexed entities. 

#### Retrieving the results

All indexed domain classes provides .search() method which lets you to list the results.
The plugin provides a search DSL for simplifying the way you can search. Here is what it looks like with the search DSL:
(See the HibernateSearchQueryBuilder class to check the available methods)

```groovy
class SomeController {

   def myAction = { MyCommand command ->

      def page = [max: Math.min(params.max ? params.int('max') : 10, 50), offset: params.offset ? params.int('offset') : 0]

      def myDomainClasses = MyDomainClass.search().list {

         if ( command.dateTo ) {
            below "publishedDate", command.dateTo
         }

         if ( command.dateFrom ) {
            above "publishedDate", command.dateFrom
         }

         mustNot {
            keyword "status", Status.DISABLED
         }

         if ( command.keyword ) {
            should {
               command.keyword.tokenize().each { keyword ->

                  def wild = keyword.toLowerCase() + '*'

                  wildcard "author", wild
                  wildcard "body", wild
                  wildcard "summary", wild
                  wildcard "title", wild
                  wildcard "categories.name", wild
               }
            }
         }

         sort "publishedDate", "asc"

         maxResults page.max

         offset page.offset
      }

      [myDomainClasses: myDomainClasses]
   }
}
```

#### Mixing with criteria query

Criteria criteria = fullTextSession.createCriteria( clazz ).createAlias("session", "session").add(Restrictions.eq("session.id", 115L));

```groovy
  def myDomainClasses = MyDomainClass.search().list {
    
    criteria {
       setFetchMode("authors", FetchMode.JOIN)
    }
    
    fuzzy "description", "mi search"
  }
```

#### Sorting the results

sort() method accepts an optional second parameter to specify the sort order: "asc"/"desc". Default is "asc".

```groovy
MyDomainClass.search().list {
   ...
   sort "publishedDate", "asc"
   ...  
}
```

If for some reasons, you want to sort results with a property which doesn't exist in your domain class, you should specify the sort type with a third parameter (default is String). You have three ways to achieve this:

##### By Specifying the type (could be Integer, String, Date, Double, Float, Long, Bigdecimal):

```groovy
MyDomainClass.search().list {
   ...
   sort "my_special_field", "asc", Integer
   ...
}
```

##### By Specifying directly its sort field (Lucene):

```groovy
def items = Item.search().list {
  ...
  sort "my_special_field", "asc", org.apache.lucene.search.SortField.Type.STRING_VAL
  ...
}
```

##### By specifying its sort field with string:

```groovy
def items = Item.search().list {
  ...
  sort "my_special_field", "asc", "string_val"
  ...
}
```


#### Support for ignoreAnalyzer(), ignoreFieldBridge() and boostedTo() functions

When searching for data, you may want to not use the field bridge or the analyzer. All methods (below, above, between, keyword, fuzzy) accept an optional map parameter to support this:

```groovy

MyDomainClass.search().list {

   keyword "status", Status.DISABLED, [ignoreAnalyzer: true]
   
   wildcard "description", "hellow*", [ignoreFieldBridge: true, boostedTo: 1.5f]
   
}
```

##### Fuzzy search

On fuzzy search, you can add an optional parameter to specify the threshold limit

```groovy

MyDomainClass.search().list {

   keyword "status", Status.DISABLED, [ignoreAnalyzer: true]

   fuzzy "description", "hellow", [ignoreFieldBridge: true, threshold: 0.8f]

}
```

#### Counting the results

You can also retrieve the number of results by using 'count' method:

```groovy
def myDomainClasses = MyDomainClass.search().count {
 ...
}
```


#### Support for projections

Hibernate Search lets you to return only a subset of properties rather than the whole domain object. It makes it possible to avoid to query the database. This plugin supports this feature:

```groovy
def myDomainClasses = MyDomainClass.search().list {

    projection "author", "body"

}

myDomainClasses.each { result ->

    def author = result[0]
    def body  = result[1]

    ...
}

```


Don't forget to store the properties into the index as following:

```groovy
class MyDomainClass {

    [...]

    static search = {
        author index: 'yes', store: 'yes'
        body index: 'yes', store: 'yes'
    }
}
```


### Analysis

#### Define named analyzers

Named analyzers are global and can be defined within application.groovy as following:

```groovy

import org.apache.solr.analysis.StandardTokenizerFactory
import org.apache.solr.analysis.LowerCaseFilterFactory
import org.apache.solr.analysis.NGramFilterFactory

...

grails.plugins.hibernatesearch = {

    analyzer( name: 'ngram', tokenizer: StandardTokenizerFactory ) {
        filter LowerCaseFilterFactory
        filter factory: NGramFilterFactory, params: [minGramSize: 3, maxGramSize: 3]
    }

}

```

This configuration is strictly equivalent to this annotation configuration:

```java
@AnalyzerDef(name = "ngram", tokenizer = @TokenizerDef(factory = StandardTokenizerFactory.class),
  filters = {
    @TokenFilterDef(factory = LowerCaseFilterFactory.class),
    @TokenFilterDef(factory = NGramFilterFactory.class, 
      params = {
        @Parameter(name = "minGramSize",value = "3"),
        @Parameter(name = "maxGramSize",value = "3") 
     })
})
public class Address {
...
}
```

#### Use named analyzers

Set the analyzer at the entity level: all fields will be indexed with the analyzer

```groovy
class MyDomainClass {

    String author
    String body
    ...

    static search = {
        analyzer = 'ngram'
        author index: 'yes'
        body index: 'yes'
    }

}
```

Or set the analyzer at the field level: 

```groovy
class MyDomainClass {

    String author
    String body
    ...

    static search = {
        author index: 'yes'
        body index: 'yes', analyzer: 'ngram'
        other index: 'yes', analyzer: new MyFilter()
    }

}
```

#### Get scoped analyzer for given entity

The plugin lets you ro retrieve the scoped analyzer for a given analyzer with the search() method:

```groovy
def parser = new org.apache.lucene.queryParser.QueryParser (
    "title", Song.search().getAnalyzer() )
```

### Filters

#### Define named filters

Named filters are global and can be defined within application.groovy as following:

```groovy

...

grails.plugins.hibernatesearch = {

    // cf official doc http://docs.jboss.org/hibernate/stable/search/reference/en-US/html_single/#query-filter
    // Example 5.20. Defining and implementing a Filter
    fullTextFilter name: "bestDriver", impl: BestDriversFilter

    // cf official doc http://docs.jboss.org/hibernate/stable/search/reference/en-US/html_single/#query-filter
    // Example 5.21. Creating a filter using the factory pattern    
    fullTextFilter name: "security", impl: SecurityFilterFactory, cache: "instance_only"
    
}

```


#### Filter query results

Filter query results looks like this:


MyDomainClass.search().list {


```groovy

// without params:
MyDomainClass.search().list {
  ...
  filter "bestDriver"
  ...
}

// with params:
MyDomainClass.search().list {
  ...
   filter name: "security", params: [ level: 4 ]
  ...
}

```


If you don't want to define named filter within application.groovy, you can also filter results as following:

```groovy
MyDomainClass.search().list {
  ...
  filter = new BestDriversFilter()
  ...
}
```

### Options 

```groovy
grails.plugins.hibernatesearch = {
	rebuildIndexOnStart false // see related section above
	throwOnEmptyQuery false // throw or not exception when Hibernate Search raises an EmptyQueryException
	fullTextFilter /* ... */ // see related section above
}
```

## Examples
A sample project is available at this repository URL
https://github.com/lgrignon/grails3-quick-start

It contains several branches for each version of this plugin

## Change log

### v2.1.2
* Supports hibernate.configClass if any
* Removed dependencies to info.app.grailsVersion, info.app.name

### v2.1 
* Grails 3.2.x
* GORM 6
* Hibernate 5.2.9
* Hibernate Search 5.7

### v2.0.2
Support for indexing trait properties

### v2.0.1 
Support for indexing inherited properties

### v2.0 
* Grails 3.1.x
* GORM 5
* Hibernate 5.1.1
* Hibernate Search 5.5.4

### v1.x 
* Grails 2.x
* Hibernate 4

## Authors

**Mathieu Perez**

+ http://twitter.com/mathieuperez

**Julie Ingignoli**

+ http://twitter.com/ZeJulie

**Louis Grignon**

+ https://github.com/lgrignon

## Development / Contribution

Install with:
```
gradlew clean publishToMavenLocal
```


Publish with:
```
gradlew clean bintrayUpload --stacktrace -PbintrayUser=... -PbintrayKey=...
```

## License

Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0
