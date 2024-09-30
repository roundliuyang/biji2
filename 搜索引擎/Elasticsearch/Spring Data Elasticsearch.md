# Spring Data Elasticsearch



## **1. Overview**

In this tutorial, **we’ll explore the basics of Spring Data Elasticsearch** in a code-focused and practical manner.

We’ll learn how to index, search, and query Elasticsearch in a Spring application using Spring Data Elasticsearch. Spring Data Elasticseach is a Spring module that implements Spring Data, thus offering a way to interact with the popular open-source, Lucene-based search engine.

**While Elasticsearch can work without a hardly defined schema, it’s still a common practice to design one and create mappings specifying the type of data we expect in certain fields**. When a document is indexed, its fields are processed according to their types. For example, a text field will be tokenized and filtered according to mapping rules. We can also create filters and tokenizers of our own.

For the sake of simplicity, we’ll use a docker image for our Elasticsearch instance, though **any Elasticsearch instance listening on port 9200 will do**.

We’ll start by firing up our Elasticsearch instance:

```bash
docker run -d --name es762 -p 9200:9200 -e "discovery.type=single-node" elasticsearch:7.6.2
```

## **2. Spring Data**

Spring Data helps avoid boilerplate code. For example, if we define a repository interface that extends the *ElasticsearchRepository* interface that Spring Data Elasticsearch provides*,* CRUD operations for the corresponding document class will become available by default.

Additionally, method implementations will generate for us simply by declaring methods with names in a predefined format. There’s no need to write an implementation of the repository interface.

The Baeldung guides on [Spring Data](https://www.baeldung.com/spring-data) provide the essentials to get started on this topic.

### **2.1. Maven Dependency** 

Spring Data Elasticsearch provides a Java API for the search engine. In order to use it, we need to add a new dependency to the *pom.xml*:

```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-elasticsearch</artifactId>
    <version>4.0.0.RELEASE</version>
</dependency>Copy
```

### **2.2. Defining Repository Interfaces** 

In order to define new repositories, we’ll extend one of the provided repository interfaces, replacing the generic types with our actual document and primary key types.

It’s important to note that *ElasticsearchRepository* extends from *PagingAndSortingRepository.* This allows built-in support for pagination and sorting.

In our example, we’ll use the paging feature in our custom search methods:

```java
public interface ArticleRepository extends ElasticsearchRepository<Article, String> {

    Page<Article> findByAuthorsName(String name, Pageable pageable);

    @Query("{\"bool\": {\"must\": [{\"match\": {\"authors.name\": \"?0\"}}]}}")
    Page<Article> findByAuthorsNameUsingCustomQuery(String name, Pageable pageable);
}
```

With the *findByAuthorsName* method, the repository proxy will create an implementation based on the method name. The resolution algorithm will determine that it needs to access the *authors* property, and then search the *name* property of each item.

The second method, *findByAuthorsNameUsingCustomQuery*, uses a custom Elasticsearch boolean query defined using the *@Query* annotation, which requires strict matching between the author’s name and the provided *name* argument.



### **2.3. Java Configuration**

When configuring Elasticsearch in our Java application, we need to define how we connect to the Elasticsearch instance. For that, we’ll extend *AbstractElasticsearchConfiguration* class:

```java
@Configuration
@EnableElasticsearchRepositories(basePackages = "com.baeldung.spring.data.es.repository")
@ComponentScan(basePackages = { "com.baeldung.spring.data.es.service" })
public class Config extends AbstractElasticsearchConfiguration {

    @Bean
    @Override
    public RestHighLevelClient elasticsearchClient() {
        ClientConfiguration clientConfiguration = ClientConfiguration.builder()
            .connectedTo("localhost:9200")
            .build();

        return RestClients.create(clientConfiguration)
            .rest();
    }
}

```

We’re using a standard Spring-enabled style annotation. *@EnableElasticsearchRepositories* will make Spring Data Elasticsearch scan the provided package for Spring Data repositories.

In order to communicate with our Elasticsearch server, we’ll use a simple *[RestHighLevelClient](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/current/java-rest-high.html)**.* While Elasticsearch provides multiple types of clients, using the *RestHighLevelClient* is a good way to future-proof the communication with the server.

*ElasticsearchOperations* bean needed to execute operations on our server is already provided by base class.

## **3. Mappings**

We use mappings to define a schema for our documents. By defining a schema for our documents, we protect them from undesired outcomes, such as mapping to an unwanted type.

Our entity is a simple document, *Article,* where the *id* is of the type *String*. We’ll also specify that such documents must be stored in an index named *blog* within the *article* type.

```java
@Document(indexName = "blog", type = "article")
public class Article {

    @Id
    private String id;
    
    private String title;
    
    @Field(type = FieldType.Nested, includeInParent = true)
    private List<Author> authors;
    
    // standard getters and setters
}
```

Indexes can have several types, which we can use to implement hierarchies.

We’ll mark the *authors* field as *FieldType.Nested*. This allows us to define the *Author* class separately, but still have the individual instances of author embedded in an *Article* document when it’s indexed in Elasticsearch.

## **4. Indexing Documents** 

Spring Data Elasticsearch generally auto-creates indexes based on the entities in the project. However, we can also create an index programmatically via the operations template:

```java
elasticsearchOperations.indexOps(Article.class).create();
```

Then we can add documents to the index:

```java
Article article = new Article("Spring Data Elasticsearch");
article.setAuthors(asList(new Author("John Smith"), new Author("John Doe")));
articleRepository.save(article);Copy
```



## **5. Querying**

### **5.1. Method Name-Based Query**

When we use the method name-based query, we write methods that define the query we want to perform. During the setup, Spring Data will parse the method signature and create the queries accordingly:

```java
String nameToFind = "John Smith";
Page<Article> articleByAuthorName
  = articleRepository.findByAuthorsName(nameToFind, PageRequest.of(0, 10));Copy
```

By calling *findByAuthorsName* with a *PageRequest* object, we’ll obtain the first page of results (page numbering is zero-based), with that page containing at most 10 articles. The page object also provides the total number of hits for the query, along with other handy pagination information.



### **5.2. A Custom Query**

There are a couple of ways to define custom queries for Spring Data Elasticsearch repositories. One way is to use the *@Query* annotation, as demonstrated in section 2.2.

Another option is to use the query builder to create our custom query.

If we want to search for articles that have the word “*data*” in the title, we can just create a *NativeSearchQueryBuilder* with a Filter on the *title:*

```java
Query searchQuery = new NativeSearchQueryBuilder()
   .withFilter(regexpQuery("title", ".*data.*"))
   .build();
SearchHits<Article> articles = 
   elasticsearchOperations.search(searchQuery, Article.class, IndexCoordinates.of("blog");
```

## **6. Updating and Deleting** 

In order to update a document, we must first retrieve it:

```java
String articleTitle = "Spring Data Elasticsearch";
Query searchQuery = new NativeSearchQueryBuilder()
  .withQuery(matchQuery("title", articleTitle).minimumShouldMatch("75%"))
  .build();

SearchHits<Article> articles = 
   elasticsearchOperations.search(searchQuery, Article.class, IndexCoordinates.of("blog");
Article article = articles.getSearchHit(0).getContent();Copy
```

Then we can make changes to the document by editing the content of the object using its assessors:

```java
article.setTitle("Getting started with Search Engines");
articleRepository.save(article);
```

As for deleting, there are several options. We can retrieve the document and delete it using the *delete* method:

```java
articleRepository.delete(article);
```

We can also delete it by *id* once we know it:

```java
articleRepository.deleteById("article_id");
```

It’s also possible to create custom *deleteBy* queries and make use of the bulk delete feature offered by Elasticsearch:

```java
articleRepository.deleteByTitle("title");
```

## **7. Conclusion**

In this article, we explored how to connect and make use of Spring Data Elasticsearch. We discussed how to query, update, and delete documents. Finally, we learned how to create custom queries if what’s offered by Spring Data Elasticsearch doesn’t fit our needs.

As usual, the source code used throughout this article can be found [over on GitHub](https://github.com/eugenp/tutorials/tree/master/persistence-modules/spring-data-elasticsearch).