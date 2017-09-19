---
layout: post
title: Testing Microservices — Java & Spring Boot
tags: programming testing java
excerpt: Based on the previous post about testing microservices, I'll show how to implement different types of tests for a Spring Boot microservice written  in Java
comments: true
---

In my [previous post](/blog/testing-microservices/) I give a round-trip of what it means to test microservices. We looked at the test pyramid and found out that you should write different types of automated tests to come up with a reliable and effective test suite.

While the previous post was more abstract this post will be more hands on and include code, lots of code. We will explore how we can implement the concepts discussed before. The technology of choice for this post will be **Java** with **Spring Boot** as the application framework. Most of the tools and libraries outlined here work for Java in general and don't require you to use Spring Boot at all. A few of them are test helpers specific to Spring Boot. Even if you don't use Spring Boot for your application there will be a lot to learn for you.

{% include table_of_contents.html %}

## Tools and Libraries We'll Look at
This post will demonstrate several tools and libraries that help us implement automated tests. The most important ones are:

  * [**JUnit**](http://junit.org) as our test runner
  * [**Mockito**](http://site.mockito.org/) for mocking dependencies
  * [**Wiremock**](http://wiremock.org/) for stubbing out third-party services
  * [**MockMVC**](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-mvc-tests) for writing HTTP integration tests (this one's Spring specific)
  * [**Pact**](https://docs.pact.io/) for writing CDC tests
  * [**Selenium**](http://docs.seleniumhq.org/) for writing UI-driven end-to-end tests
  * [**REST-assured**](https://github.com/rest-assured/rest-assured) for writing REST API-driven end-to-end tests

## The Sample Application
I've written a [simple microservice](https://github.com/hamvocke/spring-testing) including a test suite with tests for the different layers of the test pyramid. There are more tests than necessary for an application of this size. The tests on different levels overlap. This actively contradicts the advice that you should avoid test duplication throughout your test pyramid. Here I decided to go for duplication for demonstration purposes. Please keep in mind that this is not what you want for your real-world application. Duplicated tests are smelly and will be more annoying than helpful in the long term.

The sample application shows traits of a typical microservice. It provides a REST interface, talks to a database and fetches information from a third-party REST service. It's implemented in [Spring Boot ](https://projects.spring.io/spring-boot/) and should be understandable even if you've never worked with Spring Boot before.

Make sure to check out [the code on GithHub](https://github.com/hamvocke/spring-testing). The readme contains instructions you need to run the application and its automated tests on your machine.

### Functionality
The application's functionality is simple. It provides a REST interface with three endpoints:

  1. `GET /hello`: Returns _"Hello World"_. Always.
  2. `GET /hello/{lastname}`: Looks up the person with the provided last name. If the person is known, returns _"Hello {Firstname} {Lastname}"_.
  3. `GET /weather`: Returns the current weather conditions for _Hamburg, Germany_.

### High-level Structure
On a high-level the system has the following structure:

![sample application structure](/assets/img/uploads/testService.png)
_the high level structure of our microservice system_

Our microservice provides a REST interface that can be called via HTTP. For some endpoints the service will fetch information from a database. In other cases the service will call an external [weather API](https://darksky.net) via HTTP to fetch and display current weather conditions.

### Internal Architecture
Internally, the Spring Service has a Spring-typical architecture:

![sample application architecture](/assets/img/uploads/testArchitecture.png)
_the internal structure of our microservice_

  * `Controller` classes provide _REST_ endpoints and deal with _HTTP_ requests and responses
  * `Repository` classes interface with the _database_ and take care of writing and reading data to/from persistent storage
  * `Client` classes talk to other APIs, in our case it fetches _JSON_ via _HTTPS_ from the darksky.net weather API
  * `Domain` classes capture our [domain model](https://en.wikipedia.org/wiki/Domain_model) including the domain logic (which, to be fair, is quite trivial in our case).

Experienced Spring developers might notice that a frequently used layer is missing here: Inspired by [Domain-Driven Design](https://en.wikipedia.org/wiki/Domain-driven_design) a lot of developers build a **service layer** consisting of _service_ classes. I decided not to include a service layer in this application. One reason is that our application is simple enough, a service layer would have been an unnecessary level of indirection. The other one is that I think people overdo it with service layers. I often encounter codebases where the entire business logic is captured within service classes. The domain model becomes merely a layer for data, not for behaviour (Martin Fowler calls this an [Aenemic Domain Model](https://en.wikipedia.org/wiki/Anemic_domain_model)). For every non-trivial application this wastes a lot of potential to keep your code well-structured and testable and does not fully utilize the power of object orientation.

Our repositories are straightforward and provide simple <abbr title="Create Read Update Delete">CRUD</abbr> functionality. To keep the code simple I used [Spring Data](http://projects.spring.io/spring-data/). Spring Data gives us a simple and generic CRUD repository implementation that we can use instead of rolling our own. It also takes care of spinning up an in-memory database for our tests instead of using a real PostgreSQL database as it would in production.

Take a look at the codebase and make yourself familiar with the internal structure. It will be useful for our next step: Testing the application!

## Unit Tests
Unit tests have the narrowest scope of all the tests in your test suite. Depending on the language you're using (and depending on who you ask) unit tests usually test single functions, methods or classes. Since we're working in Java, an object-oriented language, our unit tests will test methods in our Java classes. A good rule of thumb is to have one test class per class of production code.

### What to Test?
The good thing about unit tests is that you can write them for all your production code classes, regardless of their functionality or which layer in your internal structure they belong to. You can unit tests controllers just like you can unit test repositories, domain classes or file readers. Simply stick to the **one test class per production class** rule of thumb and you're off to a good start.

A unit test class should at least **test the _public_ interface of the class**. Private methods can't be tested anyways since you simply can't call them from a different test class. _Protected_ or _package-private_ are accessible from a test class (given the package structure of your test class is the same as with the production class) but testing these methods could already go too far.

There's a fine line when it comes to writing unit tests: They should ensure that all your non-trivial code paths are tested (including happy path and edge cases). At the same time they shouldn't be tied to your implementation too closely.

Why's that?

Tests that are too close to the production code quickly become annoying. As soon as you refactor your production code (quick recap: refactoring means changing the internal structure of your code without changing the externally visible behavior) your unit tests will break.

This way you lose one big benefit of unit tests: acting as a safety net for code changes. You rather become fed up with those stupid tests failing every time you refactor, causing more work than being helpful and whose idea was this stupid testing stuff anyways?

What do you do instead? Don't reflect your internal code structure within your unit tests. Test for observable behavior instead. Think about

> _"if I enter values `x` and `y`, will the result be `z`?"_

instead of

> _"if I enter `x` and `y`, will the method call class A first, then call class B and then return the result of class A plus the result of class B?"_

Private methods should generally be considered an implementation detail that's why you shouldn't even have the urge to test them.

I often hear opponents of unit testing (or <abbr title="Test-Driven Development">TDD</abbr>) arguing that writing unit tests becomes pointless work where you have to test all your methods in order to come up with a high test coverage. They often cite scenarios where an overly eager team lead forced them to write unit tests for getters and setters and all other sorts of trivial code in order to come up with 100% test coverage.

There's so much wrong with that.

Yes, you should _test the public interface_. More importantly, however, you **don't test trivial code**. You won't gain anything from testing simple _getters_ or _setters_ or other trivial implementations (e.g. without any conditional logic). Save the time, that's one more meeting you can attend, hooray! Don't worry, [Kent Beck said it's ok](https://stackoverflow.com/questions/153234/how-deep-are-your-unit-tests/).

### But I _Really_ Need to Test This Private Method
If you ever find yourself in a situation where you _really really_ need to test a private method you should take a step back and ask yourself why.

I'm pretty sure this is more of a design problem than a scoping problem. Most likely you feel the need to test a private method because it's complex and testing this method through the public interface of the class requires a lot of awkward setup.

Whenever I find myself in this situation I usually come to the conclusion that the class I'm testing is already too complex. It's doing too much and violates the _single responsibility_ principle -- the _S_ of the five [_SOLID_](https://en.wikipedia.org/wiki/SOLID_(object-oriented_design)) principles.

The solution that often works for me is to split the original class into two classes. It often only takes one or two minutes of thinking to find a good way to cut the one big class into two smaller classes with individual responsibility. I move the private method (that I urgently want to test) to the new class and let the old class call the new method. Voilà, my awkward-to-test private method is now public and can be tested easily. On top of that I have improved the structure of my code by adhering to the single responsibility principle.

### Test Structure
A good structure for all your tests (this is not limited to unit tests) is this one:

  1. Set up the test data
  2. Call your method under test
  3. Assert that the expected results are returned

There's a nice mnemonic to remember this structure: [_"Arrange, Act, Assert"_](http://wiki.c2.com/?ArrangeActAssert). Another one that you can use takes inspiration from <abbr title="Behavior-Driven Development">BDD</abbr>. It's the _"given"_, _"when"_, _"then"_ triad, where _given_ reflects the setup, _when_ the method call and _then_ the assertion part.

This pattern can be applied to other, more high-level tests as well. In every case they ensure that your tests remain easy and consistent to read. On top of that tests written with this structure in mind tend to be shorter and more expressive.

### An Example
Now that we know what to test and how to structure our unit tests we can finally see a real example.

Let's take a simplified version of the `ExampleController` class:

{% highlight java %}
@RestController
public class ExampleController {

    private final PersonRepository personRepo;

    @Autowired
    public ExampleController(final PersonRepository personRepo) {
        this.personRepo = personRepo;
    }

    @GetMapping("/hello/{lastName}")
    public String hello(@PathVariable final String lastName) {
        Optional<Person> foundPerson = personRepo.findByLastName(lastName);

        return foundPerson
                .map(person -> String.format("Hello %s %s!",
		    person.getFirstName(),
		    person.getLastName()))
                .orElse(String.format("Who is this '%s' you're talking about?", lastName));
    }
}
{% endhighlight %}

A unit test for the `hello(lastname)` method could look like this:

{% highlight java %}
public class ExampleControllerTest {

    private ExampleController subject;

    @Mock
    private PersonRepository personRepo;

    @Before
    public void setUp() throws Exception {
        initMocks(this);
        subject = new ExampleController(personRepo);
    }

    @Test
    public void shouldReturnFullNameOfAPerson() throws Exception {
        Person peter = new Person("Peter", "Pan");
        given(personRepo.findByLastName("Pan"))
            .willReturn(Optional.of(peter));

        String greeting = subject.hello("Pan");

        assertThat(greeting, is("Hello Peter Pan!"));
    }

    @Test
    public void shouldTellIfPersonIsUnknown() throws Exception {
        given(personRepo.findByLastName(anyString()))
            .willReturn(Optional.empty());

        String greeting = subject.hello("Pan");

        assertThat(greeting, is("Who is this 'Pan' you're talking about?"));
    }
}
{% endhighlight %}

We're writing the unit tests using [JUnit](http://junit.org), the de-facto standard testing framework for Java. We use [Mockito](http://site.mockito.org/) to replace the real `PersonRepository` class with a stub for our test. This stub allows us to define canned responses the stubbed method should return in this test. Stubbing makes our test more simple, predictable and allows us to easily setup test data.

Following the _arrange, act, assert_ structure, we write two unit tests -- a positive case and a case where the searched person cannot be found. The first, positive test case creates a new person object and tells the mocked repository to return this object when it's called with _"Pan"_ as the value for the `lastName` parameter. The test then goes on to call the method that should be tested. Finally it asserts that the response is equal to the expected response.

The second test works similarly but tests the scenario where the tested method does not find a person for the given parameter.

## Integration Tests
Integration tests are the next higher level in your test pyramid. They test that your application can successfully integrate with its sorroundings (databases, network, filesystems, etc.). For your automated tests this means you don't just need to run your own application but also the component you're integrating with. If you're testing the integration with a database you need to run a database when running your tests. For testing that you can read files from a disk you need to save a file to your disk and use it as load it in your integration test.

### What to Test?
A good way to think about where you should have integration tests is to think about all places where data gets serialized or deserialized. Common ones are:

  * reading HTTP requests and sending HTTP responses through your REST API
  * reading and writing from/to a database
  * reading and writing from/to a filesystem
  * sending HTTP(S) requests to other services and parsing their responses

In the sample codebase you can find integration tests for `Repository`, `Controller` and `Client` classes. All these classes interface with the sorroundings of the application (databases or the network) and serialize and deserialize data. We can't test these integrations with unit tests.

### Database Integration
The `PersonRepository` is the only repository class in the codebase. It relies on _Spring Data_ and has no actual implementation. It just extends the `CrudRepository` interface and provides a single method header. The rest is Spring magic.

{% highlight java %}
public interface PersonRepository extends CrudRepository<Person, String> {
    Optional<Person> findByLastName(String lastName);
}
{% endhighlight %}

With the `CrudRepository` interface Spring Boot offers a fully functional CRUD repository with `findOne`, `findAll`, `save`, `update` and `delete` methods. Our custom method definition (`findByLastName()`) extends this basic functionality and gives us a way to fetch `Person`s by their last name. Spring Data analyses the return type of the method and its method name and checks the method name against a naming convention to figure out what it should do.

Although Spring Data does the heavy lifting of implementing database repositories I still wrote a database integration test. You might argue that this is _testing the framework_ and something that I should avoid as it's not our code that we're testing. Still, I believe having at least one integration test here is crucial. First it tests that our custom `findByLastName` method actually behaves as expected. Secondly it proves that our repository used Spring's magic correctly and can connect to the database.

To make it easier for you to run the tests on your machine (without having to install a PostgreSQL database) our test connects to an in-memory _H2_ database.

I've defined H2 as a test dependency in the `build.gradle` file. The `application.properties` in the test directory doesn't define any `spring.datasource` properties. This tells Spring Data to use an in-memory database. As it finds H2 on the classpath it simply uses H2 when running our tests.

When running the real application with the `int` profile (e.g. by setting `SPRING_PROFILES_ACTIVE=int` as environment variable) it connects to a PostgreSQL database as defined in the `application-int.properties`.

I know, that's an awful lot of Spring magic to know and understand. To get there, you'll have to sift through [a lot of documentation](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-sql.html#boot-features-embedded-database-support). The resulting code is easy on the eye but hard to understand if you don't know the fine details of Spring.

On top of that going with an in-memory database is risky business. After all, our integration tests run against a different type of database than they would in production. Go ahead and decide for yourself if you prefer Spring magic and simple code over an explicit yet more verbose implementation.

Enough explanation already, here's a simple integration test that saves a Person to the database and finds it by its last name:

{% highlight java %}
@RunWith(SpringRunner.class)
@DataJpaTest
public class PersonRepositoryIntegrationTest {
    @Autowired
    private PersonRepository subject;

    @After
    public void tearDown() throws Exception {
        subject.deleteAll();
    }

    @Test
    public void shouldSaveAndFetchPerson() throws Exception {
        Person peter = new Person("Peter", "Pan");
        subject.save(peter);

        Optional<Person> maybePeter = subject.findByLastName("Pan");

        assertThat(maybePeter, is(Optional.of(peter)));
    }
}
{% endhighlight %}

You can see that our integration test follows the same _arrange, act, assert_ structure as the unit tests. Told you that this was a universal concept!

### REST API Integration
Testing our microservice's REST API is quite simple. Of course we can write simple unit tests for all `Controller` classes and call the controller methods directly as a first measure. `Controller` classes should generally be quite straightforward and focus on request and response handling. Avoid putting business logic into controllers, that's none of their business (_best pun ever..._). This makes our unit tests straightforward (or even unnecessary, if it's too trivial).

As Controllers make heavy use of [Spring MVC's](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html) annotations for defining endpoints, query parameters and so on we won't get very far with unit tests. We want to see if our API works as expected: Does it have the correct endpoints, interpret input parameters and answer with correct HTTP status codes and response bodies? To do so, we have to go beyond unit tests.

One way to test our API were to start up the entire Spring Boot service and fire real HTTP requests against our API. With this approach we were on the very top of our test pyramid. Luckily there's another, a little less end-to-end way.

Spring MVC comes with a nice testing utility we can use: With [MockMVC](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-testing-spring-boot-applications-testing-autoconfigured-mvc-tests)we can spin up a small slice of our spring application, use a <abbr title="Domain-Specific Language">DSL</abbr> to fire test requests at our API and check that the returned data is as expected.

Let's see how this works for the `/hello/<lastname>` endpoint `ExampleController`:

{% highlight java %}
@RestController
public class ExampleController {
    private final PersonRepository personRepository;

    // shortened for clarity

    @GetMapping("/hello/{lastName}")
    public String hello(@PathVariable final String lastName) {
        Optional<Person> foundPerson = personRepository.findByLastName(lastName);

        return foundPerson
             .map(person -> String.format("Hello %s %s!", person.getFirstName(), person.getLastName()))
             .orElse(String.format("Who is this '%s' you're talking about?", lastName));
    }
}
{% endhighlight %}

Our controller calls the `PersonRepository` in the `/hello/<lastname>` endpoint. For our tests we need to replace this repository class with a mock to avoid hitting a real database. Even though this is an integration test, we're testing the REST API integration, not the database integration. That's why we stub the database in this case. The controller integration test looks as follows:

{% highlight java %}
@RunWith(SpringRunner.class)
@WebMvcTest(controllers = ExampleController.class)
public class ExampleControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private PersonRepository personRepository;

    // shortened for clarity

    @Test
    public void shouldReturnFullName() throws Exception {
        Person peter = new Person("Peter", "Pan");
        given(personRepository.findByLastName("Pan")).willReturn(Optional.of(peter));

        mockMvc.perform(get("/hello/Pan"))
                .andExpect(content().string("Hello Peter Pan!"))
                .andExpect(status().is2xxSuccessful());
    }
}
{% endhighlight %}

I annotated the test class with `@WebMvcTest` to tell Spring which controller we're testing. This mechanism instructs Spring to only start the Rest API slice of our application. We won't hit any repositories so spinning them up and requiring a database to connect to would simply be wasteful.

Instead of relying on the real `PersonRepository` we replace it with a mock in our Spring context using the `@MockBean` annotation. This annotation replaces the annotated class with a Mockito mock globally, all classes that are `@Autowired` will only find the `@MockBean` in the Spring context and wire that one instead of a real one. In our test methods we can set the behaviour of these mocks exactly as we would in a unit test, it's a Mockito mock after all.

To use `MockMvc` we can simply `@Autowire` a MockMvc instance. In combination with the `@WebMvcTest` annotation this is all Spring needs to fire test requests against our controller and expect return values and HTTP status codes. The `MockMVC` DSL is quite powerful and gets you a long way. Fiddle around with it to see what else you can do.

### Integration With Third-Party Services
Our microservice talks to [darksky.net](https://darksky.net), a weather REST API. Of course we want to ensure that our service sends requests and parses the responses correctly.

We want to avoid hitting the real _darksky_ servers when running automated tests. Quota limits of our free plan is only part of the reason. The real reason is _decoupling_. Our tests should run independently of whatever the lovely people at darksky.net are doing. Even when your machine can't access the _darksky_ servers (e.g. when you're coding on the airplane again instead of enjoying being crammed into a tiny airplane seat) or the darksky servers are down for some reason.

We can avoid hitting the real _darksky_ servers by running our own, fake _darksky_ server while running our integration tests. This might sound like a huge task. Thanks to tools like [Wiremock](http://wiremock.org/) it's easy peasy. Watch this:

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest
public class WeatherClientIntegrationTest {

    @Autowired
    private WeatherClient subject;

    @Rule
    public WireMockRule wireMockRule = new WireMockRule(8089);

    @Test
    public void shouldCallWeatherService() throws Exception {
        wireMockRule.stubFor(get(urlPathEqualTo("/some-test-api-key/53.5511,9.9937"))
                .willReturn(aResponse()
                        .withBody(FileLoader.read("classpath:weatherApiResponse.json"))
                        .withHeader(CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
                        .withStatus(200)));

        Optional<WeatherResponse> weatherResponse = subject.fetchWeather();

        Optional<WeatherResponse> expectedResponse = Optional.of(new WeatherResponse("Rain"));
        assertThat(weatherResponse, is(expectedResponse));
    }
}
{% endhighlight %}

To use Wiremock we instanciate a `WireMockRule` on a fixed port (`8089`). Using the DSL we can set up the Wiremock server, define the endpoints it should listen on and set canned responses it should respond with.

Next we call the method we want to test, the one that calls the third-party service and check if the result is parsed correctly.

It's important to understand how the test knows that it should call the fake Wiremock server instead of the real _darksky_ API. The secret is in our `application.properties` file contained in `src/test/resources`. This is the properties file Spring loads when running tests. In this file we override configuration like API keys and URLs with values that are suitable for our testing purposes, e.g. calling the the fake Wiremock server instead of the real one:

    weather.url = http://localhost:8089

Note that the port defined here has to be the same we define when instanciating the `WireMockRule` in our test. Replacing the real weather API's URL with a fake one in our tests is made possible by injecting the URL in our `WeatherClient` class' constructor:

{% highlight java %}
@Autowired
public WeatherClient(final RestTemplate restTemplate,
                     @Value("${weather.url}") final String weatherServiceUrl,
                     @Value("${weather.api_key}") final String weatherServiceApiKey) {
    this.restTemplate = restTemplate;
    this.weatherServiceUrl = weatherServiceUrl;
    this.weatherServiceApiKey = weatherServiceApiKey;
}
{% endhighlight %}

This way we tell our `WeatherClient` to read the `weatherUrl` parameter's value from the `weather.url` property we define in our application properties.

### Parsing and Writing JSON
Writing a REST API these days you often pick JSON when it comes to sending your data over the wire. Using Spring there's no need to writing JSON by hand nor to write logic that transforms your objects into JSON (although you can do both if you feel like reinventing the wheel). Defining <abbr title="Plain Old Java Object">POJOs</abbr> that represent the JSON structure you want to parse from a request or send with a response is enough.

Spring and [Jackson](https://github.com/FasterXML/jackson) take care of everything else. With the help of Jackson, Spring automagically parses JSON into Java objects and vice versa. If you have good reasons you can use any other JSON mapper out there in your codebase. The advantage of Jackson is that it comes bundled with Spring Boot.

Spring often hides the parsing and converting to JSON part from you as a developer. If you define a method in a `RestController` that returns a POJO, Spring MVC will automatically convert that POJO to a JSON string and put it in the response body. With Spring's `RestTemplate` you get the same magic. Sending a request using `RestTemplate` you can provide a POJO class that should be used to parse the response. Again it's Jackson being used under the hood.

When we talk to the weather API we receive a JSON response. The `WeatherResponse` class is a POJO representation of that JSON structure including all the fields we care about (which is only `response.currently.summary`). Using the `@JsonIgnoreProperties(ignoreUnknown = true)` annotation on our POJO objects gives us a [tolerant reader](https://www.martinfowler.com/bliki/TolerantReader.html), an interface that is liberal in what data it accepts (following [Postel's Law](https://en.wikipedia.org/wiki/Robustness_principle)). This way there can be all kinds of silly stuff in the JSON response we receive from the weather API. As long as `response.currently.summary` is there, we're happy.

If you want to test-drive your Jackson Mapping take a look at the `WeatherResponseTest`. This one tests the conversion of JSON into a `WeatherResponse` object. Since this deserialization is the only conversion we do in the application there's no need to test if a `WeatherResponse` can be converted to JSON correctly. Using the approach outlined below it's very simple to test serialization as well, though.

{% highlight java %}
@Test
public void shouldDeserializeJson() throws Exception {
   String jsonResponse = FileLoader.read("classpath:weatherApiResponse.json");
   WeatherResponse expectedResponse = new WeatherResponse("Rain");

   WeatherResponse parsedResponse = new ObjectMapper().readValue(jsonResponse, WeatherResponse.class);

   assertThat(parsedResponse, is(expectedResponse));
}
{% endhighlight %}

In this test case I read a sample JSON response from a file and let Jackson parse this JSON response using `ObjectMapper.readValue()`. Then I compare the result of the conversion with an expected `WeatherResponse` to see if the conversion works as expected.

You can argue that this kind of test is rather a unit than an integration test. Nevertheless, this kind of test can be pretty valuable to make sure that your JSON serialization and deserialization works as expected. Having these tests in place allows you to keep the integration tests around your REST API and your client classes smaller as you don't need to check the entire JSON conversion again.

## CDC Tests
Consumer-Driven Contract (CDC) tests ensure that both parties involved in an interface between two services (the provider and the consumer) stick to the  defined interface contract. This way contract tests ensure that the integration between two services remains intact.

Writing CDC tests can be as easy as sending HTTP requests to a deployed version of the service we're integrating against and verifying that the service answers with the expected data and status codes. Rolling your own CDC tests from scratch is straightforward but will soon send you down a rabbit hole. All of a sudden you need come up with a way to bundle our CDC tests, distribute them between teams and find a way to do versioning. While this is certainly possible, I want to demonstrate a different way.

In this example I'm using [Pact](https://github.com/DiUS/pact-jvm) to implement the consumer and provider side of our CDC tests.

Pact is available for multiple languages and can therefore also be used in a polyglot context. Using Pact we only need to exchange JSON files between consumers and providers. One of the more advanced features even gives us a so called ["pact broker"](https://github.com/pact-foundation/pact_broker/tree/master) that we can use to exchange pacts between teams and show which services integrate with each other.

Contract tests always include both sides of an interface -- the consumer and the provider. Both parties need to write and run automated tests to ensure that their changes don't break the interface contract. Let's see what either side has to do when using Pact.

### Consumer Test (our end)
Our microservice consumes the weather API. So it's our responsibility to write a **consumer test** that defines our expectations for the contract (the API) between our microservice and the weather service.

First we include a library for writing pact consumer tests in our `build.gradle`:

    testCompile('au.com.dius:pact-jvm-consumer-junit_2.11:3.5.5')

Thanks to this library we can implement a consumer test and use pact's mock services:

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest
public class WeatherClientConsumerTest {

    @Autowired
    private WeatherClient weatherClient;

    @Rule
    public PactProviderRuleMk2 weatherProvider = new PactProviderRuleMk2("weather_provider", "localhost", 8089, this);

    @Pact(consumer="test_consumer")
    public RequestResponsePact createPact(PactDslWithProvider builder) throws IOException {
        return builder
                .given("weather forecast data")
                .uponReceiving("a request for a weather request for Hamburg")
                    .path("/some-test-api-key/53.5511,9.9937")
                    .method("GET")
                .willRespondWith()
                    .status(200)
                    .body(FileLoader.read("classpath:weatherApiResponse.json"), ContentType.APPLICATION_JSON)
                .toPact();
    }

    @Test
    @PactVerification("weather_provider")
    public void shouldFetchWeatherInformation() throws Exception {
        Optional<WeatherResponse> weatherResponse = weatherClient.fetchWeather();
        assertThat(weatherResponse.isPresent(), is(true));
        assertThat(weatherResponse.get().getSummary(), is("Rain"));
    }
}
{% endhighlight %}

If you look closely, you'll see that the `WeatherClientConsumerTest` is very similar to the `WeatherClientIntegrationTest`. Instead of using Wiremock for the server stub we use Pact this time. In fact the consumer test works exactly as the integration test, we replace the real third-party server with a stub, define the expected response and check that our client can parse the response correctly. The difference is that the consumer test generates a **pact file** (found in `target/pacts/<pact-name>.json`) each time it runs. This pact file describes our expectations for the contract in a special JSON format.

You see that this is where the **consumer-driven** part of CDC comes from. The consumer drives the implementation of the interface by describing their expectations. The provider has to make sure that they fulfill all expectations and they're done. No gold-plating, no YAGNI and stuff.

We can take the pact file and hand it to the team providing the interface. They in turn can take this pact file and write a provider test using the expectations defined in there. This way they test if their API fulfills all our expectations.

Getting the pact file to the providing team can happen in multiple ways. A simple one is to check them into version control and tell the provider team to always fetch the latest version of the pact file. A more advances one is to use an artifact repository, a service like Amazon's S3 or the pact broker. Start simple and grow as you need.

In your real-world application you don't need both, an _integration test_ and a _consumer test_ for a client class. The sample codebase contains both to show you how to use either one. If you want to write CDC tests using pact I recommend sticking to the latter. The effort of writing the tests is the same. Using pact has the benefit that you automatically get a pact file with the expectations to the contract that other teams can use to easily implement their provider tests. Of course this only makes sense if you can convince the other team to use pact as well. If this doesn't work, using the _integration test_ and Wiremock combination is a decent plan b.

### Provider Test (the other team)
The provider test has to be implemented by the people providing the weather API. We're consuming a public API provided by darksky.net. In theory the darksky team would implement the provider test on their end to check that they're not breaking the contract between their application and our service.

Obviously they don't care about our meager sample application and won't implement a CDC test for us. That's the big difference between a public-facing API and an organisation adopting microservices. Public-facing APIs can't consider every single consumer out there or they'd become unable to move forward. Within your own organisation, you can -- and should. Your app will most likely serve a handful, maybe a couple dozen of consumers max. You'll be fine writing provider tests for these interfaces in order to keep a stable system.

The providing team gets the pact file and runs it against their providing service. To do so they implement a provider test that reads the pact file, stubs out some test data and runs the expectations defined in the pact file against their service.

The pact folks have written several libraries for implementing provider tests. Their main [GitHub repo](https://github.com/DiUS/pact-jvm) gives you a nice overview which consumer and which provider libraries are available. Pick the one that best matches your tech stack.

For simplicity let's assume that the darksky API is implemented in Spring Boot as well. In this case they could use the [Spring pact provider](https://github.com/DiUS/pact-jvm/tree/master/pact-jvm-provider-spring) which hooks nicely into Spring's MockMVC mechanisms. A hypothetical provider test that the darksky.net team would implement could look like this:

{% highlight java %}
@RunWith(RestPactRunner.class)
@Provider("weather_provider") // same as in the "provider_name" part in our clientConsumerTest
@PactFolder("target/pacts") // tells pact where to load the pact files from
public class WeatherProviderTest {
    @InjectMocks
    private ForecastController forecastController = new ForecastController();

    @Mock
    private ForecastService forecastService;

    @TestTarget
    public final MockMvcTarget target = new MockMvcTarget();

    @Before
    public void before() {
        initMocks(this);
        target.setControllers(forecastController);
    }

    @State("weather forecast data") // same as the "given()" part in our clientConsumerTest
    public void weatherForecastData() {
        when(forecastService.fetchForecastFor(any(String.class), any(String.class)))
                .thenReturn(weatherForecast("Rain"));
    }
}
{% endhighlight %}

You see that all the provider test has to do is to load a pact file (e.g. by using the `@PactFolder` annotation to load previously downloaded pact files) and then define how test data for pre-defined states should be provided (e.g. using Mockito mocks). There's no custom test to be implemented. These are all derived from the pact file. It's important that the provider test has matching counterparts to the _provider name_ and _state_ declared in the consumer test.

I know that this whole CDC thing can be confusing as hell when you get started. Believe me when I say it's worth taking your time to understand it. If you need a more thorough example, go and check out the [fantastic example](https://github.com/lplotni/pact-example) my friend [Lukasz](https://twitter.com/lplotni) has written. This repo demonstrates how to write consumer and provider tests using pact. It even features both Java and JavaScript services so that you can see how easy it is to use this approach with different programming languages.

## End-to-End Tests
At last we arrived at top of our test pyramid (phew, almost there!). Time to write end-to-end tests that calls our service via the user interface and does a round-trip through the complete system.

### Using Selenium (testing via the UI)
For end-to-end tests [Selenium](http://docs.seleniumhq.org/) and the [WebDriver](https://www.w3.org/TR/webdriver/) protocol are the tool of choice for many developers. With Selenium you can pick a browser you like and let it automatically call your website, click here and there, enter data and check that stuff changes in the user interface.

Selenium needs a browser that it can start and use for running its tests. There are multiple so-called _'drivers'_ for different browsers that you could use. [Pick one](https://www.mvnrepository.com/search?q=selenium+driver) (or multiple) and add it to your `build.gradle`:

    testCompile('org.seleniumhq.selenium:selenium-firefox-driver:3.5.3')

Running a fully-fledged browser in your test suite can be a hassle. Especially when using continuous delivery the server running your pipeline might not be able to spin up a browser including a user interface (e.g. because there's no X-Server available). You can take a workaround for this problem by starting a virtual X-Server like [xvfb](https://en.wikipedia.org/wiki/Xvfb).

A more recent approach is to use a _headless_ browser (i.e. a browser that doesn't have a user interface) to run your webdriver tests. Until recently [PhantomJS](http://phantomjs.org/) was the leading headless browser used for browser automation. Ever since both [Chromium](https://developers.google.com/web/updates/2017/04/headless-chrome) and [Firefox](https://developer.mozilla.org/en-US/Firefox/Headless_mode) announced that they've implemented a headless mode in their browsers PhantomJS all of a sudden became obsolete. After all it's better to test your website with a browser that your users actually use (like Firefox and Chrome) instead of using an artificial browser just because it's convenient for you as a developer.

Both, headless Firefox and Chrome, are brand new and yet to be widely adopted for implementing webdriver tests. We want to keep things simple. Instead of fiddling around to use the bleeding edge headless modes let's stick to the classic way using Selenium and a regular browser. A simple end-to-end test that fires up Firefox, navigates to our service and checks the content of the website looks like this:

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class HelloE2ESeleniumTest {

    private WebDriver driver = new FirefoxDriver();

    @LocalServerPort
    private int port;

    @After
    public void tearDown() {
        driver.close();
    }

    @Test
    public void helloPageHasTextHelloWorld() {
        driver.get(String.format("http://127.0.0.1:%s/hello", port));

        assertThat(driver.findElement(By.tagName("body")).getText(), containsString("Hello World!"));
    }
}
{% endhighlight %}

Note that this test will only run on your system if you have Firefox installed on the system you run this test on (your local machine, your CI server).

The test is straightforward. It spins up the entire Spring application on a random port using `@SpringBootTest`. We then instanciate a new Firefox webdriver, tell it to go navigate to the `/hello` endpoint of our microservice and check that it prints "Hello World!" on the browser window. Cool stuff!

### Using RestAssured (Testing via the REST API)
I know, we already have tests in place that fire some sort of request against our REST API and check that the results are correct. Still, none of them is truly end to end. The MockMVC tests are "only" integration tests and don't send real HTTP requests against a fully running service.

Let me show you one last tool that can come in handy when you write a service that provides a REST API. [REST-assured](https://github.com/rest-assured/rest-assured) is a library that gives you a nice DSL for firing real HTTP requests against an API and checks the responses. It looks similar to MockMVC but is truly end-to-end (fun fact: there's even a REST-Assured MockMVC dialect). If you think Selenium is overkill for your application as you don't really have a user interface that needs testing, REST-Assured is the way to go.

First things first: Add the dependency to your `build.gradle`.

    testCompile('io.rest-assured:rest-assured:3.0.3')

With this library at our hands we can implement a end-to-end test for our REST API:

{% highlight java %}
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class HelloE2ERestTest {

    @Autowired
    private PersonRepository personRepository;

    @LocalServerPort
    private int port;

    @After
    public void tearDown() throws Exception {
        personRepository.deleteAll();
    }

    @Test
    public void shouldReturnGreeting() throws Exception {
        Person peter = new Person("Peter", "Pan");
        personRepository.save(peter);

        when()
                .get(String.format("http://localhost:%s/hello/Pan", port))
        .then()
                .statusCode(is(200))
                .body(containsString("Hello Peter Pan!"));
    }
}
{% endhighlight %}

Again, we start the entire Spring application using `@SpringBootTest`. In this case we `@Autowire` the `PersonRepository` so that we can write test data into our database easily. When we now ask the REST API to say "hello" to our friend "Mr Pan" we're being presented with a nice greeting. Amazing! And more than enough of an end-to-end test if you don't even sport a web interface.

## Some Advice Before You Go
There we go, you made it through the entire testing pyramid. Congratulations! Before you go, there are some more general pieces of advice that I think will be helpful on your journey. Keep these in mind and you'll soon write automated tests that truly kick ass:

  1. Test code is as important as production code. Give it the same level of care and attention. Never allow sloppy code to be justified with the _"this is only test code"_ claim
  2. Test one condition per test. This helps you to keep your tests short and easy to reason about
  3. _"arrange, act, assert"_ or _"given, when, then"_ are good mnemonics to keep your tests well-structured
  4. Readability matters. Don't try to be overly <abbr title="Don't Repeat Yourself">DRY</abbr>. Duplication is okay, if it improves readability. Try to find a balance between [DRY and <abbr title="Descriptive and Meaningful Phrases">DAMP</abbr>](https://stackoverflow.com/questions/6453235/what-does-damp-not-dry-mean-when-talking-about-unit-tests) code
  5. When in doubt use the [Rule of Three](https://blog.codinghorror.com/rule-of-three/) to decide when to refactor. _Use before reuse_.

Now it's your turn. Go ahead and make sure your microservices are properly tested. Your life will be more relaxed and your features will be written in almost no time. Promise!
