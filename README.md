EXTERNAL WEBSITE INSTRUCTIONS

https://maikkingma.github.io/clean-hexagonal-onion-docs/docs/setting-up-an-initial-application

1: Workshop Intro
Welcome to the Clean Hexagonal Onion with a Dash of DDD in Spring workshop at DevBcn 2023. My name is Maik Kingma, and I will be your coach for today. Since I cannot be with everyone simultaneously, these docs will provide you with a step-by-step guide.

Getting Help
I will do my best to answer all the questions. We will have a few sync moments throughout the workshop, where we can discuss questions together. Keep validating your work after every chapter. In case you get stuck somewhere along the way, you can always continue the workshop on a code base snapshot from one of the prepared branches that provide you the solution of each chapter.

Time management
There are 7 implementation chapters in total. They become increasingly complex. Today we only have 2 hours, let's see how far we get. The workshop will remain online so you can continue at home as well. At the end of each section, there is a git branch reference with which you can continue to the next chapter, in case you got stuck.

Example:
if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: XXX")}
else{
    log.error("Shout for help!") || (git stash && git checkout 5-persist-author-data-done)
}

What you will need
JDK 17
Docker Desktop
An IDE of your choice (ideally IntelliJ)
an Internet connection
TDD
If you want there is the possibility to do Test Driven Development. At the end of each chapter you will find a happy flow unit test, in some cases also an integration test. You can use these to do TDD. If you want an even more guided approach, I prepared TDD branches for each chapter in the repository. Ideally, only use those if you are entirely stuck.

Side note
The purpose of this workshop is to get to know the Clean Hexagonal Onion pattern and how to apply it in combination with some DDD in a Spring Boot application. I do not intend to have full test coverage in this project. Instead, the provided unit-tests usually only test the happy flow, paired with an occasional integration test or two.


2: Check out the Spring service
Setting up the project from scratch would take up too much time and focus away from the important content of this workshop. Hence, I took the liberty of preparing a repository that you can fork or clone.

Button to GitHub is located in the header menu. Alternatively, click here.

You can either check out the prepared setup branch or, optionally set the project up yourself.

git checkout setup-done

If you checked out this branch then you can proceed to the validation step and skip the following OPTIONAL steps. (there are still some mandatory steps after the optional ones). If you chose to set up the project yourself, please follow the OPTIONAL steps.

OPTIONAL: DO IT YOURSELF
If you feel like you want to do it all yourself you can use Spring Initializr to setup our spring boot project.

Go to the Spring Initializr
Choose your Project Dependency Manager of your choice (Maven or Gradle).
Choose your Language of choice.
For the Spring Boot version at least 3.0.7 (3.1.0 is preselected).
Fill in the Project Metadata as you see fit, but make sure packaging is Jar and the Java version is 17.
On the right-hand side you can see the add dependencies button. You need to add the following:
Spring Web
Spring Data JPA
PostgreSQL Driver
Liquibase Migration
Lombok
Spring Boot DevTools (optional)
OPTIONAL: Your config should be something like this:
spring-initializr.png

OPTIONAL: Create the clean hexagonal onion folder structure
package-structure.png

OPTIONAL: Database
add these lines to src/main/resources/application.properties

# DataSource
spring.datasource.url=jdbc:postgresql://localhost:5432/clean-hexagonal-onion-service
spring.datasource.username=postgres
spring.datasource.password=postgres

and create a ./docker-compose.yml file and paste this content into it:

version: '3.9'

services:
  postgres:
    image: postgres
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: clean-hexagonal-onion-service
    ports:
      - '5432:5432'
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

Validate
Run the docker-compose script from the IDE or from the terminal with docker compose up --build. Last but not least, in order to avoid errors for now, please comment out the liquibase dependency in your ./pom.xml.

Note: You may have to enable annotation processing in your IDE for the Lombok dependency.

Validate that the Spring application starts by running the application (without any errors). similar to this: package-structure.png

Alternatively, run:

mvn clean verify


3: Getting to know the domain
We will be building a simple service, where we can register book authors, let them write books and eventually publish them. The publishers though will not be part of our bounded context, and we will need to retrieve any information about them from an external publisher API (https://github.com/MaikKingma/publisher-service).

In our domain model we have identified two aggregates: Author and Book. A publisher is information from another bounded context. It might change over time, and we have no control over it as such. That is why we make it an immutable value object in our context, that is not persisted to our database. The only thing we need to know from a publisher, is the identifier of the publisher. We will need it in order to identify the publisher of a book.

Domain Diagram
domain-model.png

The API
POST /authors/commands/register
POST /authors/{id}/commands/writeBook
GET /authors

GET /books?title
POST /books/{id}/commands/publish

If there are any questions about the domain model or the API definition feel free to ask anybody from codecentric or your neighbour.


4: Create the register Author command
For this step you will need to create classes and an interface in two different packages in our project.

Command
We need to create a REST endpoint that allows us to register Authors.

### Register an author
POST /authors/commands/register HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "firstName": "PLACE_YOUR_FIRST_NAME",
  "lastName": "PLACE_YOUR_LAST_NAME"
}

This endpoint should create an Author from the given Data Transfer Object (DTO) (or payload) and call the register function of an author on the domain layer service (a port / interface). The standard response should be empty with status code 202.

Tip: You can add the above snippet in a file ./http/AuthorCommands.http which allows you to execute the request from inside your IDE (if supported, IntelliJ does).

Domain Core
According to the domain model we need to create the class /domain/author/Author.java in our domain package. To keep control of the creation of our aggregates we make the all args constructor private and instead create a factory method public static Author createAuthor(String fristName, String lastName). Only our factory method will access the private all args constructor. That way we can keep control of the creation of Authors at all times outside our domain package. For now that is all we need. No getters, no setters or builders needed for now. In case you are asking yourself: And what about the id? Rest assured! We will solve this one later. It remains null for now.

author.png

The data port
In order to be able to interact with our domain, we need to define a port (interface) called /domain/author/AuthorService.java in our domain package. The required members can be found in the domain model. Afterwards, we inject this into our AuthorCommands.javaclass in the constructor (you could autowire it but let's stick to constructor injection).

Nice to know: this complies with the SOLID principle of 'dependency inversion'. Good for us :)

author-service.png

The Data Layer
No injection without at least one Spring Bean implementing the interface. After having injected the interace, at least INtelliJ will show a warning or error that it requires at least one implementation.In /data/author/AuthorServiceImpl.java we implement /domain/author/AuthorService.java and annotate it with the @Service annotation from Spring. For now, simply add a log statement of your choice to the implementation of the method void registerAuthor(Author author).

Validation
Let's test our code. Feel free to write your own test. Alternatively, copy and paste this test class into your project and run it. All should be green :-).

@SpringBootTest
@AutoConfigureMockMvc
class AuthorCommandsTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void register() throws Exception {
        //given
        var registerAuthorPayloadJson = objectMapper.writeValueAsString(new RegisterAuthorPayload("firstName", "lastName"));

        //when //then
        mockMvc.perform(post("/authors/commands/register")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(registerAuthorPayloadJson))
                .andExpect(status().isAccepted());
    }
}


OPTIONAL: Run the app on localhost
By the way, if you run the docker compose file ./docker-compose.yml and start the Spring app you can also test your API at runtime manually. Got to http/AuthorCommands.http and run the request against your localhost:8080.

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: **Persisting Data**.")}
else{
    log.error("Shout for help!") || (git stash && git checkout 4-create-author-command-done)
}




5: Persist the Author data to the DB
For this step you will need to create classes and an interface in the data source section of the "clean hexagonal onion". Also, we need to create a DB migration script with liquibase to enable the JPA for our Author entity

Liquibase
Uncomment the liquibase dependency in ./pom.xml

<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
    <version>3.10.3</version>
</dependency>

and add these lines to src/main/resources/application.properties

# DB Migration
spring.liquibase.change-log=classpath:db/db.changelog-master.xml

create the file src/main/resources/db/db.changelog-master.xml with content

<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    <include file="changelog/01_create_author_seq.xml" relativeToChangelogFile="true"/>
    <include file="changelog/02_create_author_table.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>


create the file src/main/resources/db/changelog/01_create_author_seq.xml with content

<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

    <changeSet id="01-create-author-sequence" author="Maik Kingma">
        <comment>Create Author sequence</comment>
        <createSequence sequenceName="author_seq" minValue="10001"/>
    </changeSet>
</databaseChangeLog>


create file src/main/resources/db/changelog/02_create_author_table.xml with content

<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

    <changeSet id="01-create-author-table" author="Maik Kingma">
        <comment>Create author table</comment>

        <createTable tableName="author">
            <column name="id" type="bigint">
                <constraints primaryKey="true" primaryKeyName="author_id_pk" nullable="false"/>
            </column>
            <column name="first_name" type="text">
                <constraints nullable="false"/>
            </column>
            <column name="last_name" type="text">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>


Persistence with JPA
Following the clear segregation of duties in our "clean hexagonal onion", we need to create a number of classes to achieve persistence. This can seem overkill at first, but remember, DDD is not for small POCs or start up projects. It is designed to solve complexity for bigger projects, and so is our "clean hexagonal onion". We value simplicity, clarity and maintainability over cleverness and complexity.

We will need to create a JPA entity for our Author, a mapper class that maps from domain to JPA, and an actual JPA repository interface that links our entity to the DB.

JPA entity
create a class /data/AuthorJPA.java annotated with

@Entity
@Builder
@NoArgsConstructor()
@AllArgsConstructor
@Table(name = "author")

Fill this class with the same fields as the Author.java class. By adding this JPA class, which is independent of the domain aggregate, we keep the domain only loosely coupled to the domain core. Usually, what we often see in Spring applications is that the domain aggregate is the data entity and domain entity/aggregate at the same time.

Having created that class, we now need to teach our AuthorJPA data model where to get the id from in the DB. We have already created the liquibase script that generates an author_seq for us. We annotate our id field with the following:

@Id
@GeneratedValue(strategy = SEQUENCE, generator = "author_seq_gen")
@SequenceGenerator(name = "author_seq_gen", sequenceName = "author_seq", allocationSize = 1)

JPA repository
create an interface /data/AuthorRepository.java that extends JPARepository<AuthorJPA, Long> and annotate the interface with @Repository

JPA to domain Mapper
Since we want to decouple our data model from the actual domain model, we need to teach our application how to map between domain and data model. There are fancy libraries for this such as mapstruct but let us code it ourselves for now.

Create a class /data/AuthorMapper.java with a method that maps all fields of Author.class to the corresponding fields of AuthorJPA.class and returns the instance of it. Method signature:

public static AuthorJPA mapToJPA(Author author)

Hint: You may need to add some getters. And you can make use of the builder pattern.

Updating the AuthorServiceImpl.java
In the previous task we only added a log statement to the AuthorServiceImpl.registerAuthor implementation. Inject the AuthorRepository into the AuthorServiceImpl and update the function in such a way that the Author is persisted to the DB.

Validation
Let's test your implementation. Update the AuthorCommandsIntegrationTest.java and replace the previously added register() test with the following Test:

    @BeforeEach
    void beforeAll() {
        authorRepository.deleteAll();
    }

    @Test
    void registerAndGet() throws Exception {
        //given
        var registerAuthorPayloadJson = objectMapper.writeValueAsString(new RegisterAuthorPayload("firstName", "lastName"));
        var expected = AuthorJPA.builder().firstName("firstName").lastName("lastName").build();
        //when
        mockMvc.perform(post("/authors/commands/register")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(registerAuthorPayloadJson))
                .andExpect(status().isAccepted());
        authorRepository.flush();
        // then
        assertThat(authorRepository.findAll().size()).isEqualTo(1);
        assertThat(authorRepository.findAll().get(0)).usingRecursiveComparison().ignoringFields("id").isEqualTo(expected);
    }


To not affect our runtime DB with changes from our unit tests, we need to complete two steps before running this test suite:

update the pom.xml with the following dependency:
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>

add the test config file src/test/resources/application.properties with content
# DataSource
spring.datasource.driver-class-name=org.h2.Driver
spring.datasource.url=jdbc:h2:mem:db;DB_CLOSE_DELAY=-1
spring.datasource.username=sa
spring.datasource.password=sa

# DB Migration
spring.liquibase.change-log=classpath:db/db.changelog-master.xml

Now we can finally run the test. Well done!

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: Querying our Domain.")}
else{
    log.error("Shout for help!") || (git stash && git checkout 5-persist-author-data-done)
}


6: Query Authors
What goes up must come down!

or in our case

What goes into the DB must come out!

OPTIONAL: Run the app on localhost
By the way, if you run your docker compose file and start the Spring app you can also test your API at runtime manually. Got to http/AuthorCommands.http and run the request against your localhost:8080.

(in case you don't have it, as it is part of my part of prepared *-done branches)

### Register an author
POST /authors/commands/register HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "firstName": "PLACE_YOUR_FIRST_NAME",
  "lastName": "PLACE_YOUR_LAST_NAME"
}

Implement the Query endpoint
We want to make our authors readable via our REST API. For that purpose we introduce

### Get authors
GET /authors HTTP/1.1
Host: localhost:8080
Accept: application/json

This chapter will be a bit less guided in terms of code snippets. Let's see how far you get!

You need to create a GET mapping in a new file /query/AuthorQueries.java in our query section of the clean hexagonal onion. Remember, that we also need to decouple the query layer from our domain core. Hence, we introduce the following view model that our query will return:
public record AuthorView (Long id, String name) {
  public AuthorView(Author author) {
    this(author.getId(), author.getFullName());
  }
}

Update the AuthorService.java (and in turn also the AuthorServiceImpl.java)
Update the AuthorMapper.java because we now need to map from data model to domain model. (add Builders and Getters where necessary)
Hint 1: The annotation @Builder(builderMethodName = "restore") might come in quite useful in the Author.class

Hint 2: You may want to try TDD to complete this one :-)

Validate
Let's test your implementation:

@SpringBootTest
@AutoConfigureMockMvc
class AuthorQueriesTest {

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @Autowired
  private EntityManager entityManager;

  @BeforeEach
  void beforeAll() {
    entityManager.createNativeQuery("DELETE FROM author WHERE true;").executeUpdate();
  }

  @Test
  @Transactional
  void getAll() throws Exception {
    // given
    var authorJPA = AuthorJPA.builder().firstName("firstName").lastName("lastName").build();
    entityManager.persist(authorJPA);
    entityManager.flush();
    AuthorView expected = new AuthorView(Author.createAuthor("firstName", "lastName"));
    // when then
    MvcResult result = mockMvc.perform(get("/authors")
                    .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andReturn();

    var resultingAuthorViews = objectMapper.readValue(
            result.getResponse().getContentAsString(), new TypeReference<List<AuthorView>>() { });
    assertThat(resultingAuthorViews)
            .usingRecursiveFieldByFieldElementComparatorIgnoringFields("id")
            .containsExactly(expected);
  }
}

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: Separate the Domain Interaction Layer")}
else{
    log.error("Shout for help!") || (git stash && git checkout 6-query-author-done)
}


7: Separate the Domain Interaction Layer
The keen observer might have realised already, we are missing one layer in our package structure. For simplicity in the beginning, we skipped that extra layer of segregation between our Domain Code Layer and the External Adapters Layer, consisting of Query, Command, Process, Data and ACL.

However, this is leaves us with some incorrect code, and essentially we violate the principles introduced in the Clean Hexagonal Onion concept. Can you guess where that is?

Let us have a look at the classes eu/javaland/clean_hexagonal_onion/command/author/AuthorCommands.java and eu/javaland/clean_hexagonal_onion/query/author/AuthorQueries.java. Currently, they both import the domain class Auhtor.java. And this is wrong, because we said that we want to protect our domain from the outside world and have only our domain interaction layer interact with the domain core.

This chapter is aimed at fixing that mistake.

Fixing the package structure
We start by adding the missing package domaininteraction to the package eu/javaland/clean_hexagonal_onion domain-interaction_package.png

Also, create the package eu/javaland/clean_hexagonal_onion/domaininteraction/author inside there.

Verifying what is wrong
TDD to the rescue! Here is a test that will show you what we did wrong so far. We will use ArchUnit for this:

Add the dependency to the ./pom.xml file:

<!-- Test Dependencies-->
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.0.1</version>
    <scope>test</scope>
</dependency>

Here is the test we add to src/test/java/eu/javaland/clean_hexagonal_onion/CleanHexagonalOnionArchitectureTest.java:

package nl.maikkingma.clean_hexagonal_onion;

import com.tngtech.archunit.core.importer.ImportOption;
import com.tngtech.archunit.junit.AnalyzeClasses;
import com.tngtech.archunit.junit.ArchTest;
import com.tngtech.archunit.lang.ArchRule;

import static com.tngtech.archunit.library.Architectures.layeredArchitecture;

@AnalyzeClasses(packages = "nl.maikkingma.clean_hexagonal_onion",
                importOptions = {ImportOption.DoNotIncludeTests.class})
public class CleanHexagonalOnionArchitectureTest {

    @ArchTest
    static final ArchRule layer_dependencies_are_respected =
            layeredArchitecture().consideringAllDependencies()

            .layer("command").definedBy("nl.maikkingma.clean_hexagonal_onion.command..")
            .layer("query").definedBy("nl.maikkingma.clean_hexagonal_onion.query..")
            .layer("data").definedBy("nl.maikkingma.clean_hexagonal_onion.data..")
            .layer("domain interaction").definedBy("nl.maikkingma.clean_hexagonal_onion.domaininteraction..")
            .layer("domain").definedBy("nl.maikkingma.clean_hexagonal_onion.domain..")

            .whereLayer("command").mayNotBeAccessedByAnyLayer()
            .whereLayer("query").mayNotBeAccessedByAnyLayer()
            .whereLayer("data").mayNotBeAccessedByAnyLayer()
            .whereLayer("domain interaction").mayOnlyBeAccessedByLayers("command", "query", "data")
            .whereLayer("domain").mayOnlyBeAccessedByLayers("domain interaction");
}

Note that ACL and Process are still missing in there. Since they are still empty packages our test would fail. We will add them later on.

Run the test. It will throw quite a few errors, pointing out to us the classes where we violated those dependency rules.

Moving code around
Let's start fixing things. First we move the interface eu/javaland/clean_hexagonal_onion/domain/author/AuthorService. java to eu/javaland/clean_hexagonal_onion/domaininteraction/author/AuthorService.java. Since it is essentially a port we defined to access our data source, it needs to reside in the domain interaction layer.

Looking at that interface, and considering the Single Responsibility Principle from SOLID, let us rename the interface so that it clearly states its purpose: AuthorService becomes AuthorDataService

public interface AuthorDataService {
    void save(AuthorDTO author);
    List<AuthorDTO> findAll();
}

Notice that we renamed the function registerAuthor to save here since it better describes what we are trying to achieve here. We also removed any reference to the domain core in our port and instead introduced a AuthorDTO (data transfer object) that will function as a mapping layer between the domain core and the external adapters.

Removing domain core access in the External Adapter layer
We now need to update the classes eu/javaland/clean_hexagonal_onion/command/author/AuthorCommands.java and eu/javaland/clean_hexagonal_onion/query/author/AuthorQueries.java so that they do not have to import eu/javaland/clean_hexagonal_onion/domain/author/Author.java any longer.

For that purpose we need add a Flow to the domain interaction layer that makes needed functionality available to the external adapter layer. We create the eu/javaland/clean_hexagonal_onion/domaininteraction/author/AuthorFlow.java, which will be our port for exposing the Author registration logic (in case of the command) and the finding all Authors logic (in case of query).

Let's do some TDD! Here is the test for the flow class:

package nl.maikkingma.clean_hexagonal_onion.domaininteraction.author;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class AuthorFlowTest {

    @Mock
    private AuthorDataService authorDataService;

    @InjectMocks
    private AuthorFlow authorFlow;

    @Test
    void registerAuthorByName() {
        // given
        var expectedAuthor = new AuthorDTO(null, "firstName", "lastName");
        ArgumentCaptor<AuthorDTO> argumentCaptor = ArgumentCaptor.forClass(AuthorDTO.class);
        // when
        authorFlow.registerAuthorByName("firstName", "lastName");
        // then
        verify(authorDataService, times(1)).save(argumentCaptor.capture());
        AuthorDTO actualAuthor = argumentCaptor.getValue();
        assertThat(actualAuthor).usingRecursiveComparison().isEqualTo(expectedAuthor);
    }

    @Test
    void getListOfAllAuthors() {
        // given
        var authorsData = List.of(
                new AuthorDTO(1L, "firstName1", "lastName1"),
                new AuthorDTO(2L, "firstName2", "lastName2"));
        when(authorDataService.findAll()).thenReturn(authorsData);
        // when
        List<AuthorDTO> actualResult = authorFlow.getListOfAllAuthors();
        // then
        assertThat(actualResult).containsExactlyInAnyOrder(new AuthorDTO(1L, "firstName1", "lastName1"),
                new AuthorDTO(2L, "firstName2", "lastName2"));
    }
}

Notice that we introduced an AuthorDTO object. This is the pattern we use for isolating our domain core from the outside world.

Once you are done implementing, we will need to update the command and query controllers. They should not directly access the AuthorDataService but instead make use of the AuthorFlow class in the domain interaction layer.

Splitting up the AuthorMapper
You might have realised already, our AuthorMapper does not comply either. He is located in the external adapter layer (data) but has knowledge of our domain core, namely Author.java.

Here are some tests you can add to your code in order to TDD our way out of this dilemma:

package nl.maikkingma.clean_hexagonal_onion.data.author;

import nl.maikkingma.clean_hexagonal_onion.domaininteraction.author.AuthorDTO;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AuthorJPAMapperTest {

    @Test
    void mapToJPA() {
        // given
        AuthorDTO input = new AuthorDTO(1L, "first", "last");
        AuthorJPA expectedOutput = AuthorJPA.builder()
                .id(1L)
                .firstName("first")
                .lastName("last")
                .build();
        // when
        AuthorJPA result = AuthorJPAMapper.mapToJPA(input);
        // then
        assertThat(result).usingRecursiveComparison().isEqualTo(expectedOutput);
    }
}

and

package nl.maikkingma.clean_hexagonal_onion.domaininteraction.author;

import nl.maikkingma.clean_hexagonal_onion.domain.author.Author;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class AuthorDomainMapperTest {

    @Test
    void mapToDomain() {
        // given
        var input = new AuthorDTO(1L, "first", "last");
        var expectedOutput = Author.restore()
                .id(1L)
                .firstName("first")
                .lastName("last")
                .build();
        // when
        var result = AuthorDomainMapper.mapToDomain(input);
        // then
        assertThat(result).usingRecursiveComparison().isEqualTo(expectedOutput);
    }
}

After updating the mapper structure, make sure you update the rest of the classes where necessary. Pay special attention to the classes that interact with each other across the layers. There should not be any outward facing dependencies left after completing this chapter.

Eventually, you should end up with a file tree similar to this one:

file-structure-domain-interaction.png

This was a bit of a hassle, but we are now all lined up for an even deeper dive into the clean hexagonal onion and its perks.

Well done so far!

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: Do it yourself")}
else{
    log.error("Shout for help!") || (git stash && git checkout 7-separate-domain-interaction-done)
}


8: Do it yourself
Back to school! First, you learn the theory, then you apply it to something new. What would authors be without books! (Maybe bloggers, but anyways...)

After having completed the last chapter where we split of our domain interaction layer, we need to update our domain model:

Domain Model Diagram
UPDATE DOMAIN MODEL TO CONTAIN CORRECT METHODS updated_domain_model.jpg

The API
In this chapter you will need to apply what you have learned so far and create two new commands in our API spec that will allow an author to write a book and afterwards publish it (publishing is part of the next chapter).

### DONE
POST /authors/commands/register HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "firstName": "PLACE_YOUR_FIRST_NAME",
  "lastName": "PLACE_YOUR_LAST_NAME"
}

### TODO in this assignment
POST /authors/10001/commands/writeBook HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "title": "PLACE_YOUR_TILE",
  "genre": "HORROR"
}

### TODO in this assignment
GET /books?title= HTTP/1.1
Accept: application/json
Host: localhost:8080

The assignment
Your new assignment is to enable authors to write a book. Try to find out which steps you need to take in order to do so. Try to backtrace the steps we have taken so far.

To not get hung up on Database evolution, here is the liquibase script you will need:

Update src/main/resources/db/db.changelog-master.xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    <include file="changelog/01_create_author_seq.xml" relativeToChangelogFile="true"/>
    <include file="changelog/02_create_author_table.xml" relativeToChangelogFile="true"/>
    <include file="changelog/03_create_book_seq.xml" relativeToChangelogFile="true"/>
    <include file="changelog/04_create_book_table.xml" relativeToChangelogFile="true"/>
</databaseChangeLog>


Add file changelog/03_create_book_seq.xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

    <changeSet id="03-create-book-sequence" author="Maik Kingma">
        <comment>Create Book sequence</comment>
        <createSequence sequenceName="book_seq" minValue="10001"/>
    </changeSet>
</databaseChangeLog>


Add file changelog/04_create_book_table.xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
        xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-3.6.xsd">

  <changeSet id="04-create-book-table" author="Maik Kingma">
    <comment>Create book table</comment>
    <createTable tableName="book">
      <column name="id" type="bigint">
        <constraints primaryKey="true" primaryKeyName="book_id_pk" nullable="false"/>
      </column>
      <column name="title" type="text">
        <constraints nullable="false"/>
      </column>
      <column name="author_id" type="bigint">
        <constraints nullable="false"/>
      </column>
      <column name="genre" type="text">
        <constraints nullable="false"/>
      </column>
      <column name="published" type="boolean" defaultValue="false"/>
      <column name="publisher_id" type="uuid"/>
      <column name="isbn" type="text"/>
    </createTable>

    <addForeignKeyConstraint
            baseColumnNames="author_id"
            baseTableName="book"
            constraintName="FK_AUTHOR_BOOK"
            deferrable="false"
            initiallyDeferred="false"
            onDelete="RESTRICT"
            onUpdate="RESTRICT"
            referencedColumnNames="id"
            referencedTableName="author"
            validate="true"/>
  </changeSet>
</databaseChangeLog>


Let's assume we have 4 genres available: 'FANTASY','HORROR', 'CRIME' and 'ROMANCE'. Make sure to add the enum in the code accordingly.

Now try to complete the rest!

Some things to watch our for
Remember that an entity in DDD is not the same as an entity in the ORM sense. So the OneToMany and ManyToOne relation annotations need to be placed on the JPA data model classes only.
The following code snippets might prove useful:

    @OneToMany(mappedBy = "authorJPA", fetch = FetchType.LAZY, cascade = CascadeType.ALL)
    private Set<BookJPA> books;

and

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private AuthorJPA author;

TIP: If you are stuck, TDD can be practiced with the code from the validate paragraph?

GOOD LUCK!

Validate
Done? Let's test your implementation:

the test for the creation of books:
@SpringBootTest
@AutoConfigureMockMvc
class AuthorActionCommandsTest {

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @Autowired
  private BookRepository bookRepository;

  @Autowired
  private EntityManager entityManager;

  @BeforeEach
  void beforeAll() {
    entityManager.createNativeQuery("DELETE FROM author where true; DELETE FROM book where true;")
            .executeUpdate();
  }

  @Test
  @Transactional
  void writeBook() throws Exception {
    // given
    entityManager.createNativeQuery(
                    "INSERT INTO author (id, first_name, last_name) VALUES (?,?,?)")
            .setParameter(1, 1)
            .setParameter(2, "firstName")
            .setParameter(3, "lastName")
            .executeUpdate();

    var writeBookDTOJson = objectMapper.writeValueAsString(new WriteBookPayload("title", "CRIME"));
    var expected = BookJPA.builder()
            .title("title")
            .genre("CRIME")
            .published(false)
            .author(AuthorJPA.builder().id(1L).firstName("firstName").lastName("lastName").build())
            .build();
    // when
    mockMvc.perform(post("/authors/1/commands/writeBook")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(writeBookDTOJson))
            .andExpect(status().isAccepted());
    entityManager.flush();
    // then
    List<BookJPA> books = bookRepository.findAll();
    assertThat(books.size()).isEqualTo(1);
    assertThat(books.get(0)).usingRecursiveComparison().ignoringFields("id").isEqualTo(expected);
  }
}

the test for the querying of books:
@SpringBootTest
@AutoConfigureMockMvc
class BookQueriesTest {

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @Autowired
  private EntityManager entityManager;

  @BeforeEach
  void beforeEach() {
    entityManager.createNativeQuery("DELETE FROM author where true; DELETE FROM book where true;")
            .executeUpdate();
  }

  @Test
  @Transactional
  void shouldFindBooksWithNoQueryParam() throws Exception {
    // given
    entityManager.createNativeQuery(
                    "INSERT INTO author (id, first_name, last_name) VALUES (?,?,?)")
            .setParameter(1, 1)
            .setParameter(2, "firstName")
            .setParameter(3, "lastName")
            .executeUpdate();
    var book1 = BookJPA.builder()
            .author(AuthorJPA.builder().id(1L).build())
            .genre("HORROR")
            .title("horror-book")
            .build();
    var book2 = BookJPA.builder()
            .author(AuthorJPA.builder().id(1L).build())
            .genre("ROMANCE")
            .title("romance-book")
            .build();
    var expectedBookView1 = new BookView("horror-book", "HORROR", "firstName lastName");
    var expectedBookView2 = new BookView("romance-book", "ROMANCE", "firstName lastName");

    entityManager.persist(book1);
    entityManager.persist(book2);
    entityManager.flush();
    // when
    MvcResult result = mockMvc.perform(get("/books")
                    .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andReturn();
    // then
    var resultingBookViews = objectMapper.readValue(
            result.getResponse().getContentAsString(), new TypeReference<List<BookView>>() {
            });
    assertThat(resultingBookViews).hasSize(2);
    assertThat(resultingBookViews).usingRecursiveFieldByFieldElementComparatorIgnoringFields("id")
            .containsExactlyInAnyOrder(expectedBookView1, expectedBookView2);
  }

  @Test
  @Transactional
  void shouldFindBooksFilteredByQueryParamTitle() throws Exception {
    // given
    entityManager.createNativeQuery(
                    "INSERT INTO author (id, first_name, last_name) VALUES (?,?,?)")
            .setParameter(1, 1)
            .setParameter(2, "firstName")
            .setParameter(3, "lastName")
            .executeUpdate();
    var book1 = BookJPA.builder()
            .author(AuthorJPA.builder().id(1L).build())
            .genre("HORROR")
            .title("horror-book")
            .build();
    var book2 = BookJPA.builder()
            .author(AuthorJPA.builder().id(1L).build())
            .genre("ROMANCE")
            .title("romance-book")
            .build();
    var expectedBookView = new BookView("horror-book", "HORROR", "firstName lastName");

    entityManager.persist(book1);
    entityManager.persist(book2);
    entityManager.flush();
    // when
    MvcResult result = mockMvc.perform(get("/books")
                    .accept(MediaType.APPLICATION_JSON)
                    .queryParam("title", "orror-"))
            .andExpect(status().isOk())
            .andReturn();
    // then
    var resultingBookViews = objectMapper.readValue(
            result.getResponse().getContentAsString(), new TypeReference<List<BookView>>() { });
    assertThat(resultingBookViews).hasSize(1);
    assertThat(resultingBookViews).usingRecursiveFieldByFieldElementComparatorIgnoringFields("id")
            .containsExactly(expectedBookView);
  }
}

Good job! You now know how to apply the Clean Hexagonal Onion yourself. Once again:

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: The ACL adapter")}
else{
    log.error("Shout for help!") || (git stash && git checkout 8-do-it-yourself-done)
}


9: The Anti-Corruption-Layer Adapter (ACL)
We now want to publish an author's book. In order to do so, we need to know which publishers are even available to do so. We will assume for the sake of the exercise that we need to retrieve that information from an external system outside our bounded context.

The "third-party" service
Please check out the publisher-service. It is another Spring service you will be able to retrieve the Publisher data from. The API specs are as follows:

### Get authors
GET /publishers HTTP/1.1
Host: localhost:8081
Accept: application/json

Implementation
In order to protect our domain core from (data) corruption (and our application in general) and limit the service coupling to lose coupling only, we introduce a wrapping Rest controller of our own in our service as follows:

### Get authors
GET /publishers HTTP/1.1
Host: localhost:8080
Accept: application/json

This endpoint should return a list of views of a publisher as we expect it in our bounded context. It might be that the publisher server actually returns a lot of fields their view of a publisher that we do not need at all.

Hence, implement our new API endpoint in such a way that we receive the response body in the following format:

[
  {
    "id": "55699ecc-42dc-42cf-8290-1207655140e2",
    "name": "the/experts."
  },
  {
    "id": "5e659183-411c-42af-8bed-2225a2165c59",
    "name": "Heise"
  },
  {
    "id": "8fb2e701-b086-415c-bf16-1b3096d355df",
    "name": "PubIT"
  },
  {
    "id": "42121f98-4013-42fe-8285-8fe25a783ac9",
    "name": "AwesomeBooks.nl"
  }
]

We have already created a Query adapter once in section 6. If you want more hints.... you need to create a class /query/publisher/PublisherQuery.java which contains a GetMapping for /publishers. Inject the PublisherAppService.java and call the method to retrieve all publishers via the ACL adapter.

Additionally, we will need to introduce the Publisher.java. As described in the domain model it will be a value object as it will have no lifecycle of its own within our domain (the id on the view model might be misleading, in the sense that DDD states that value objects have no ID. That ID however is only a technical identifier of another bounded context for us. It is not an ID in our context and the publisher has no further life cycle or state of its own in our system).

The data we populate the response with will come from the Publisher-service API. As mentioned before, you will need to introduce a new application service /domaininteraction/publisher/PublisherAppService.java. We add the implementation for that service in the ACL adapter layer, /acl/publisher/PublisherAppServiceImpl.java.

Hint: For the REST call execution you may want to consider this code snippet:

new RestTemplate().getForObject(uri, PublisherDTO[].class);

Give it a try! If you are stuck feel free to ask questions! And don't forget to run the PublisherService.

Validate
Let's test your implementation:

please add this line to the src/test/resources/application.properties

# Publisher service
publisher.service.host: http://localhost:${mockServerPort}

and also add these dependencies (we need no test scope on the last two as we will use them in runtime code later on)

<dependency>
    <groupId>org.mock-server</groupId>
    <artifactId>mockserver-spring-test-listener-no-dependencies</artifactId>
    <version>5.13.2</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>javax.json</groupId>
    <artifactId>javax.json-api</artifactId>
    <version>1.1.4</version>
</dependency>
<dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.json</artifactId>
    <version>1.1.4</version>
</dependency>

package nl.maikkingma.clean_hexagonal_onion.query.publisher;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.publisher.PublisherDTO;
import org.junit.jupiter.api.Test;
import org.mockserver.client.MockServerClient;
import org.mockserver.model.Header;
import org.mockserver.springtest.MockServerTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;

import javax.json.Json;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockserver.matchers.Times.exactly;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@MockServerTest
@SpringBootTest
@AutoConfigureMockMvc
class PublisherQueryTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    private MockServerClient mockServerClient;

    @Test
    void shouldGetAllPublishers() throws Exception {
        configureMockGetAllPublishers();

        var expectedPublisher1 =
                new PublisherDTO(UUID.fromString("55699ecc-42dc-42cf-8290-1207655140e2"), "the/experts");
        var expectedPublisher2 =
                new PublisherDTO(UUID.fromString("5e659183-411c-42af-8bed-2225a2165c59"), "Heise");
        var expectedPublisher3 =
                new PublisherDTO(UUID.fromString("8fb2e701-b086-415c-bf16-1b3096d355df"), "PubIT");

        // when
        MvcResult result = mockMvc.perform(get("/publishers").accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andReturn();
        // then
        var publisherList = objectMapper.readValue(
                result.getResponse().getContentAsString(), new TypeReference<List<PublisherDTO>>() {
                });
        assertThat(publisherList).hasSize(3);
        assertThat(publisherList).usingRecursiveFieldByFieldElementComparator()
                .containsExactlyInAnyOrder(expectedPublisher1, expectedPublisher2, expectedPublisher3);
    }

    private void configureMockGetAllPublishers() {
        var responseBody = Json.createArrayBuilder()
                .add(Json.createObjectBuilder()
                        .add("id", "55699ecc-42dc-42cf-8290-1207655140e2")
                        .add("name", "the/experts")
                        .add("taxNumber", "VAT12345")
                        .add("numberOfEmployees", 30)
                        .add("yearlyRevenueInMillions", 99)
                        .add("amountOfBooksPublished", 20)
                        .build())
                .add(Json.createObjectBuilder()
                        .add("id", "5e659183-411c-42af-8bed-2225a2165c59")
                        .add("name", "Heise")
                        .add("taxNumber", "VAT32723")
                        .add("numberOfEmployees", 3333)
                        .add("yearlyRevenueInMillions", 432)
                        .add("amountOfBooksPublished", 453)
                        .build())
                .add(Json.createObjectBuilder()
                        .add("id", "8fb2e701-b086-415c-bf16-1b3096d355df")
                        .add("name", "PubIT")
                        .add("taxNumber", "VAT4242111")
                        .add("numberOfEmployees", 56)
                        .add("yearlyRevenueInMillions", 21)
                        .add("amountOfBooksPublished", 4)
                        .build())
                .build().toString();

        mockServerClient.when(request().withMethod("GET").withPath("/publishers"), exactly(1)).respond(
                response()
                        .withStatusCode(200)
                        .withHeaders(new Header("Content-Type", "application/json; charset=utf-8"))
                        .withBody(responseBody)
                        .withDelay(TimeUnit.SECONDS, 1)
        );
    }
}

Hint: also, do not forget to give our ArchUnit test a run. Will it still pass?

Great job also finishing this chapter. In case you had some trouble, feel free to check out the branch below.

if (allTestsGreen == true) {
    log.info("DONE! Let's move on to the next topic: The Process adapter")}
else{
    log.error("Shout for help!") || (git stash && git checkout 9-acl-adapter-done)
}



10: The Process Adapter
We still want to publish a book. We know all available publishers from the previous section and made it available in our bounded context.

Assuming we have already created a manuscript as an author, we can now decide to publish that manuscript with one of the available publishers.

The command
For that purpose we create the /commands/book/BookCommands.java class containing our new command API endpoint:

POST /books/{id}/commands/publish HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "publisherId": "80553ae1-2ef8-4adf-8fa8-d551684a9ea3"
}

The behaviour of this endpoint should comply with the following functional requirements:

A book can only be published with a publisher that actually exits. SO, we need to validate that the chosen publisher exists in the source context of publishers, the publisher service.
We want the publishing to happen asynchronous, i.e. our command should simply return that it accepted the publishing request, and it will be passed on for further processing. (Eventual Consistency).
We achieve this passing on for further processing by publishing a domain event, you can find extra reference [here] (https://bit.ly/3Fs9Cy4).

While it looks easy at first sight, we need to invest a little more coding effort than Baeldung because we segregated the data entity from the actual domain aggregate.

1. Task: Implement the Publisher ACL adapter.
First things first: in order to validate the existence of a publisher by id, the third party publisher service from the previous section exposes the following endpoint:

### Get authors
GET /publishers/{id} HTTP/1.1
Host: localhost:8081
Accept: application/json

implement this in our ACL adapter. We need to expose some sort of interface to our later book command REST controller that we can inject.

Hint: Consider our application & domain service pattern. Since Publishers do not have a life cycle within our context, which would you choose?

2. Task: Implement the Book Command
implement our new book commands endpoint and for now only log the response of the getPublisherById call you implemented in step 1.

POST /books/{id}/commands/publish HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "publisherId": "6b4ca9c9-b3ae-4130-b3f8-2da873c3940e"
}

Hint: You can always verify that you used the correct patterns by running our eu/javaland/clean_hexagonal_onion/CleanHexagonalOnionArchitectureTest.java.

3. Task: Implement the flow
We now have a publisher that we want to publish with. Next, we need to retrieve the book corresponding to the ID form the path parameter, check that it exists, then in step 4, register it for publishing.

Following our flow pattern, we introduce the class eu/javaland/clean_hexagonal_onion/domaininteraction/publisher/PublisherFlow.java. You may ask at this point, why a PublisherFlow and not a BookFLow. This is a bit of a taste question. My reasoning is that there would be no publishing without a publisher existing, so I chose that flow. But you could equally reason, that you can only publish a book that exists. So this is eventually up to the architect to decide. Both are correct.

Here is a test for some TDD:

@ExtendWith(MockitoExtension.class)
class PublisherFlowTest {

    @Mock
    private PublisherAppService publisherAppService;

    @Mock
    private BookDataService bookDataService;

    @InjectMocks
    private PublisherFlow publisherFlow;

    @Test
    void publishBook_success() {
        // given
        UUID publisherId = UUID.randomUUID();
        var authorDTO = new AuthorDTO(1L, "firstName", "lastName");
        var bookDTO = new BookDTO(1L, authorDTO, "title", "description", null, false, null);
        var publisherDTO = new PublisherDTO(publisherId,
                "publisherName");
        when(publisherAppService.getPublisherById(publisherId.toString())).thenReturn(publisherDTO);
        when(bookDataService.findById(1L)).thenReturn(bookDTO);
        // when
        publisherFlow.publishBook(1L, publisherId.toString());
        // then
        ArgumentCaptor<BookDTO> argumentCaptor = ArgumentCaptor.forClass(BookDTO.class);
        verify(bookDataService, times(1)).save(argumentCaptor.capture());
        var capturedArg = argumentCaptor.getValue();
        assertThat(capturedArg.publisherId()).isEqualTo(publisherId);
        assertThat(capturedArg.isPublished()).isFalse();
    }

    @Test
    void publishBook_failure_publisherNotFound() {
        // given
        UUID publisherId = UUID.randomUUID();
        when(publisherAppService.getPublisherById(publisherId.toString())).thenThrow(
                new PublisherAppService.PublisherNotFoundException("Publisher not found!"));
        // when then
        assertThrows(PublisherAppService.PublisherNotFoundException.class, () -> publisherFlow.publishBook(1L,
                publisherId.toString()));
    }

    @Test
    void publishBook_failure_bookNotFound() {
        // given
        UUID publisherId = UUID.randomUUID();
        var publisherDTO = new PublisherDTO(publisherId,
                "publisherName");
        when(publisherAppService.getPublisherById(publisherId.toString())).thenReturn(publisherDTO);
        when(bookDataService.findById(1L)).thenThrow(new BookDataService.BookNotFoundException("Book not found!"));
        // when then
        assertThrows(BookDataService.BookNotFoundException.class, () -> publisherFlow.publishBook(1L,
                publisherId.toString()));
    }

    @Test
    void publishBook_failure_bookAlreadyInPublishing() {
        // given
        UUID publisherId = UUID.randomUUID();
        var authorDTO = new AuthorDTO(1L, "firstName", "lastName");
        var bookDTO = new BookDTO(1L, authorDTO, "title", "description", UUID.randomUUID(), false, null,
                new ArrayList<>());
        var publisherDTO = new PublisherDTO(publisherId,
                "publisherName");
        when(publisherAppService.getPublisherById(publisherId.toString())).thenReturn(publisherDTO);
        when(bookDataService.findById(1L)).thenReturn(bookDTO);
        // when
        assertThrows(PublisherFlow.BookAlreadyInPublishingException.class, () -> publisherFlow.publishBook(1L,
                publisherId.toString()));
    }
}


5. Task: Implement the domain event
We want to allow a publishing request and, since we aim for eventual consistency, the publishing of a related domain event, that can be consumed later on by our process adapter. To be able to handle domain events we need a new dependency in our pom.xml:

<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-commons</artifactId>
</dependency>

The challenge of this task is to actually register domain events on the domain, but then also propagate them to the actual JPA entity which will eventually be persisted by the Repository, which in turn will trigger the handling of domain events in the AbstractAggregateRoot.java. If we send domain events from the domain already, we could encounter a chicken and egg problem, since the domain event would be sent before the JPA entity is persisted.

Here are some useful snippets. Can you place them in the correct places?

    @Getter
    private final List<DomainEvent> domainEvents = new ArrayList<>();

Our Domain event

    @Value
    public static class RequestPublishingEvent extends DomainEvent {
        Long bookId;
        UUID publisherId;
    }
    // and
    public abstract class DomainEvent {}

The following snippet provides us the support of handling the Domain events correctly.

public class BookJPA extends AbstractAggregateRoot<BookJPA> {
    // ...
    public void registerDomainEvents(List<DomainEvent> domainEvents) {
      domainEvents.forEach(this::andEvent);
    }
}

In case of our domain events, we will slightly loosen our architecture rules sets and allow the process adapter to know about our domain events. This is because we want to be able to consume them in the process adapter. THe same goes for the DomainEvent.class in the data adapter. JPA needs to know about our domain events to be able to fire the events when the JPA entity is persisted. Here is an updated ArchUnit test:

@AnalyzeClasses(packages = "nl.maikkingma.clean_hexagonal_onion", importOptions = {ImportOption.DoNotIncludeTests.class})
public class CleanHexagonalOnionArchitectureTest {

    @ArchTest
    static final ArchRule layer_dependencies_are_respected =
            layeredArchitecture().consideringAllDependencies()

                    .layer("command").definedBy("nl.maikkingma.clean_hexagonal_onion.command..")
                    .layer("query").definedBy("nl.maikkingma.clean_hexagonal_onion.query..")
                    .layer("data").definedBy("nl.maikkingma.clean_hexagonal_onion.data..")
                    .layer("acl").definedBy("nl.maikkingma.clean_hexagonal_onion.acl..")
                    .layer("process").definedBy("nl.maikkingma.clean_hexagonal_onion.process..")
                    .layer("domain interaction").definedBy("nl.maikkingma.clean_hexagonal_onion.domaininteraction..")
                    .layer("domain").definedBy("nl.maikkingma.clean_hexagonal_onion.domain..")

                    .whereLayer("command").mayNotBeAccessedByAnyLayer()
                    .whereLayer("query").mayNotBeAccessedByAnyLayer()
                    .whereLayer("data").mayNotBeAccessedByAnyLayer()
                    .whereLayer("acl").mayNotBeAccessedByAnyLayer()
                    .whereLayer("process").mayNotBeAccessedByAnyLayer()
                    .whereLayer("domain interaction").mayOnlyBeAccessedByLayers("command", "query", "data", "acl", "process")
                    .whereLayer("domain").mayOnlyBeAccessedByLayers("domain interaction")
                    // we will ignore the Domain Event dependencies from the process layer to the domain layer
                    // We are eventually trying to solve complexity, not add to it. Adding another layer to solve
                    // this would be overkill and overcomplicate things
                    .ignoreDependency(EventProcessor.class, Book.RequestPublishingEvent.class)
                    .ignoreDependency(PublishBookDelegate.class, Book.RequestPublishingEvent.class)
                    .ignoreDependency(BookJPA.class, DomainEvent.class)
            ;
}


Some classes in this test are not yet implemented. We will do that in the next tasks

In the class Book.java we need to implement the method requestPublishing(id: UUID) which we previously referenced in task 4.

    public void requestPublishing(UUID publisherId) {
        // TODO assign the publisherID to the book, in our business logic this means that the book is now in the 
        // process of being published
        // TODO create a domain event and add it to the domainEvents list
    }


Having done all that, the AbstractAggregateRoot will do the publishing of the domain events for us at the moment of object persistence. So on to the next task: we need to be able to consume these events in the process adapter.

6. Task: Consume the published domain event.
Create a class /process/EventProcessor.java:

import lombok.extern.slf4j.Slf4j;
import nl.theexperts.clean_hexagonal_onion_service.domain.book.Book;
import nl.theexperts.clean_hexagonal_onion_service.process.book.PublishBookDelegate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.event.TransactionalEventListener;

import static org.springframework.transaction.event.TransactionPhase.AFTER_COMMIT;

@Slf4j
@Component
public class EventProcessor {

    private final PublishBookDelegate publishBookDelegate;

    public EventProcessor(PublishBookDelegate publishBookDelegate) {
        this.publishBookDelegate = publishBookDelegate;
    }
    
    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void handleEvent(Book.RequestPublishingEvent requestPublishingEvent) {
        log.info(requestPublishingEvent.toString());
        publishBookDelegate.publishBook(requestPublishingEvent);
    }
}

Hint: TransactionalEventListener is a Spring feature that allows you to listen to events that are published within a transaction. This is useful for publishing events that are triggered by a command. The event is only published if the transaction is committed successfully. If the transaction is rolled back, the event is not published.

This implementation will allow for the domain events on a JPA entity to be processed after the transaction was committed.

7. Task: Delegate Implementation
Now try to actually implement the /process/book/PublishBookDelegate.java class.

Useful snippet:

@Service
public class PublishBookDelegate {

  private final BookService bookService;
  private final PublisherService publisherService;


  public PublishBookDelegate(BookService bookService, PublisherService publisherService) {
    this.bookService = bookService;
    this.publisherService = publisherService;
  }

  @Transactional(propagation = REQUIRES_NEW)
  public void publishBook(Book.RequestPublishingEvent event) {
    // retrieve book by id from event
    
   // request the publishing of the book via the Publisher ACL layer (also see API docs below)

    // update the isbn of the book you received as a response and then store the book
  }
}

The API on the publisher service is defined as follows:

POST /publishers/receiveBookOffer
Host: localhost:8081
Content-Type: application/json

{
  "publisherId": "<SOME-UUID>",
  "author": "author name",
  "title": "Cool Title"
}

Returns:
{
  "isbn": "ISBN-3895b77d-ee27-40de-9b08-bf24fe2a013a"
}

Validate
Let's test your implementation:

Testing the endpoint:

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockserver.client.MockServerClient;
import org.mockserver.model.Header;
import org.mockserver.springtest.MockServerTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import javax.json.Json;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.mockserver.matchers.Times.exactly;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@MockServerTest
@SpringBootTest
@AutoConfigureMockMvc
class BookCommandsTest {

  private static final Long BOOK_ID = 1L;
  private static final Long AUTHOR_ID = 2L;

  @Autowired
  private MockMvc mockMvc;

  @Autowired
  private ObjectMapper objectMapper;

  @Autowired
  private EntityManager entityManager;

  private MockServerClient mockServerClient;

  @BeforeEach
  void beforeAll() {
    entityManager.createNativeQuery("DELETE FROM author where true; DELETE FROM book where true;")
            .executeUpdate();
  }

  @Test
  @Transactional
  void publishBook() throws Exception {
    // given
    UUID publisherUUID = UUID.randomUUID();
    configureMockGetPublisherById(publisherUUID.toString());
    entityManager.createNativeQuery(
                    "INSERT INTO author (id, first_name, last_name) VALUES (?,?,?)")
            .setParameter(1, AUTHOR_ID)
            .setParameter(2, "firstName")
            .setParameter(3, "lastName")
            .executeUpdate();

    entityManager.createNativeQuery(
                    "INSERT INTO book (id, title, author_id, genre, published, publisher_id, isbn) " +
                            "VALUES (?,?,?,?,?,?,?)")
            .setParameter(1, BOOK_ID)
            .setParameter(2, "title")
            .setParameter(3, AUTHOR_ID)
            .setParameter(4, "HORROR")
            .setParameter(5, false)
            .setParameter(6, null)
            .setParameter(7, null)
            .executeUpdate();

    entityManager.flush();

    var publishBookPayload = objectMapper.writeValueAsString(new PublishBookPayload(publisherUUID.toString()));
    // when
    mockMvc.perform(post(String.format("/books/%d/commands/publish", BOOK_ID))
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(publishBookPayload))
            .andExpect(status().isAccepted());
  }

  private void configureMockGetPublisherById(String publisherId) {
    var responseBody = Json.createObjectBuilder()
            .add("id", publisherId)
            .add("name", "the/experts")
            .add("taxNumber", "VAT12345")
            .add("numberOfEmployees", 30)
            .add("yearlyRevenueInMillions", 99)
            .add("amountOfBooksPublished", 20)
            .build().toString();

    mockServerClient.when(request().withMethod("GET").withPath("/publishers/" + publisherId), exactly(1)).respond(
            response()
                    .withStatusCode(200)
                    .withHeaders(new Header("Content-Type", "application/json; charset=utf-8"))
                    .withBody(responseBody)
                    .withDelay(TimeUnit.SECONDS,1)
    );
  }
}


Testing the event publishing: src/test/java/eu/javaland/clean_hexagonal_onion/data/book/BookJPATest.java

Note: We are wiring the actual repositories in this test scenario. We need them to handle the transactions correctly for us.

import nl.maikkingma.clean_hexagonal_onion.data.author.AuthorJPA;
import nl.maikkingma.clean_hexagonal_onion.data.author.AuthorRepository;
import nl.maikkingma.clean_hexagonal_onion.domain.book.Book;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.author.AuthorDTO;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.book.BookDTO;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.book.BookDataService;
import jakarta.persistence.EntityManager;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.ArgumentCaptor;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;
import org.springframework.transaction.event.TransactionalEventListener;

import java.util.List;
import java.util.UUID;

import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;

@SpringJUnitConfig
@SpringBootTest
class BookJPATest {

  @MockBean
  private TestEventHandler eventHandler;

  @Autowired
  private AuthorRepository authorRepository;

  @Autowired
  private BookRepository bookRepository;

  @Autowired
  private BookDataService bookDataService;

  @Autowired
  private EntityManager entityManager;

  @BeforeEach
  void beforeAll() {
    authorRepository.deleteAll();
    bookRepository.deleteAll();
  }

  @Test
  void shouldPublishEventOnSavingAggregate() {
    var authorJPA = AuthorJPA.builder().firstName("first").lastName("last").build();
    authorRepository.save(authorJPA);
    authorJPA = authorRepository.findAll().get(0);

    UUID publisherId = UUID.randomUUID();
    AuthorDTO authorDTO = new AuthorDTO(authorJPA.getId(), authorJPA.getFirstName(),
            authorJPA.getLastName());
    var bookDTO = new BookDTO(1L, authorDTO, "Title"
            , "Horror",
            publisherId,
            false,
            null, List.of(new Book.RequestPublishingEvent(1L, publisherId)));
    // when
    bookDataService.save(bookDTO);
    // then
    ArgumentCaptor<Book.RequestPublishingEvent> argumentCaptor = ArgumentCaptor.forClass(Book.RequestPublishingEvent.class);
    verify(eventHandler, times(1)).handleEvent(argumentCaptor.capture());
  }

  interface TestEventHandler {
    @TransactionalEventListener()
    void handleEvent(Book.RequestPublishingEvent event);

  }
}


Testing the event processor in src/test/.../process/EventProcessorTest.java:

import nl.theexperts.clean_hexagonal_onion_service.domain.book.Book;
import nl.theexperts.clean_hexagonal_onion_service.process.book.PublishBookDelegate;
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

import java.util.UUID;

import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;

@SpringJUnitConfig
@SpringBootTest
class EventProcessorTest {

  @Mock
  private PublishBookDelegate publishBookDelegate;

  @InjectMocks
  private EventProcessor eventProcessor;

  @Test
  void shouldCallTheDelegateToActOnEvent() {
    // when
    Book.RequestPublishingEvent requestPublishingEvent = new Book.RequestPublishingEvent(1L, UUID.randomUUID());
    eventProcessor.handleEvent(requestPublishingEvent);
    // then
    verify(publishBookDelegate, times(1)).publishBook(requestPublishingEvent);
  }
}


Testing the delegate and ACL interaction in src/test/.../process/book/PublishBookDelegateTest.java:

import nl.maikkingma.clean_hexagonal_onion.domain.book.Book;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.book.BookDTO;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.book.BookDataService;
import nl.maikkingma.clean_hexagonal_onion.domaininteraction.book.BookFlow;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import org.junit.jupiter.api.Test;
import org.mockserver.client.MockServerClient;
import org.mockserver.model.Header;
import org.mockserver.springtest.MockServerTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import javax.json.Json;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockserver.matchers.Times.exactly;
import static org.mockserver.model.HttpRequest.request;
import static org.mockserver.model.HttpResponse.response;

@MockServerTest
@SpringBootTest
class PublishBookDelegateTest {

    private static final Long BOOK_ID = 1L;
    private static final Long AUTHOR_ID = 2L;

    @Autowired
    private BookFlow bookFlow;

    @Autowired
    private BookDataService bookDataService;

    private MockServerClient mockServerClient;

    @Autowired
    private EntityManager entityManager;

    @Test
    @Transactional
    void shouldCallThePublisherServiceAPIWithCorrectPayload() {
        PublishBookDelegate publishBookDelegate = new PublishBookDelegate(bookFlow);
        UUID publisherUUID = UUID.randomUUID();
        UUID isbnUUID = UUID.randomUUID();
        configureMockPublishersReceiveBookOffer(isbnUUID.toString());

        entityManager.createNativeQuery(
                        "INSERT INTO author (id, first_name, last_name) VALUES (?,?,?)")
                .setParameter(1, AUTHOR_ID)
                .setParameter(2, "firstName")
                .setParameter(3, "lastName")
                .executeUpdate();

        entityManager.createNativeQuery(
                        "INSERT INTO book (id, title, author_id, genre, published, publisher_id, isbn) " +
                                "VALUES (?,?,?,?,?,?,?)")
                .setParameter(1, BOOK_ID)
                .setParameter(2, "title")
                .setParameter(3, AUTHOR_ID)
                .setParameter(4, "HORROR")
                .setParameter(5, false)
                .setParameter(6, null)
                .setParameter(7, null)
                .executeUpdate();

        entityManager.flush();
        BookDTO checkBook = bookDataService.findById(BOOK_ID);
        assertThat(checkBook.published()).isFalse();
        assertThat(checkBook.isbn()).isNull();
        // when
        publishBookDelegate.publishBook(new Book.RequestPublishingEvent(BOOK_ID, publisherUUID));
        // then
        mockServerClient.verify(request()
                .withPath("/publishers/receiveBookOffer")
                .withMethod("POST")
                .withBody(Json.createObjectBuilder()
                        .add("publisherId", publisherUUID.toString())
                        .add("author", "firstName lastName")
                        .add("title", "title")
                        .build().toString()));
        BookDTO resultBook = bookDataService.findById(BOOK_ID);
        assertThat(resultBook.published()).isTrue();
        assertThat(resultBook.isbn()).isEqualTo(String.format("ISBN-%s", isbnUUID));

    }

    private void configureMockPublishersReceiveBookOffer(String uuid) {
        var responseBody = Json.createObjectBuilder()
                .add("isbn", String.format("ISBN-%s", uuid))
                .build().toString();

        mockServerClient.when(request().withMethod("POST").withPath("/publishers/receiveBookOffer"), exactly(1)).respond(
                response()
                        .withStatusCode(202)
                        .withHeaders(new Header("Content-Type", "application/json; charset=utf-8"))
                        .withBody(responseBody)
                        .withDelay(TimeUnit.SECONDS,1)
        );
    }
}


Give it a try!

if (allTestsGreen == true) {
    log.info("DONE! You finished the workshop!");
else{
    log.error("Shout for help!") || (git stash && git checkout 10-process-adapter-done)
}
