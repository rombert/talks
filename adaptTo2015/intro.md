About the speaker
-------------------

* Apache Sling PMC member
* Long-time Open Source contributor
* Working with Adobe on AEM

<!--
Hello everyone and thanks for joining. My name is Robert Munteanu and this
is 'How do I test my Sling Application'..

A few words about myself - I am a PMC member in the Apache Sling project and
long-time Open Source contributor.

Also, I have been working for almost 4 years with Adobe in Romania on AEM.
So yes, it was still called CQ back then :-)

And obviously I care a lot about testing and code quality and this is why
I'm here. 
-->

Agenda
---------

* How do I test my application?
* What about Sling?
* Demo


<!--
First of all, to discuss how to test a Sling application we should quickly look
at how do we usually test an application.

Then we take these concepts and map them in detail to Apache Sling.

And in the end, should we have the time, we'll briefly look at a demo. The
slides contain a lot of code so it won't be an issue if we don't get to the
actual demo.  
-->

Speaker.getInstance().interrupt()
===


<!--
Yes, please interrupt me whenever you have question or comment to make.
-->

How do I test my application?
===

<!-- 
Now that that's done with, let's get on with the concepts about testing
applications in general.
--> 

Unit testing
--

	@Test public void isNull() { 
	  assertThat( StringUtils.isNull( null ), is(true));
	}
	
<!--
The first form of testing is of course unit testing. Unit tests should be
small, fast and self-contained. Unit tests validate that a unit - typically
a Java class works well in isolation.

This more or less applies to the 'Utils' family of classes, which don't collaborate
heavily. Which means that in projects you don't see this a lot.
-->
	
Unit testing with mocks
--
 
    @Test public void persist() {
      MyDao dao = mock(MyDao.class);
      MyService service = new MyService(dao);
      service.persist(new ServiceObject()); // must not fail
    }
    
<!--
And then comes a time when you want to test a class which has
collaborators. A typical example in a database-backed application is testing
a service which uses a data access object. This trivial example shows how it
would look like.

Two points to note here:

1. Mock-based tests work against a contract and not an implementation. In here
we expect the data access object to behave in a certain way and we specify its
behaviour in the test.

2. Mock-based tests are still small, fast and self-contained. They do tend to
become a bit more verbose than 'plain' unit tests, but are still manageable.
-->
    
Integration testing
--
 
    @Test public void persist() {
      MyDao dao = new MyRealDao(/* config */);
      MyService service = new MyService(dao);
      service.persist(newServiceObject()); // must not fail
    }    

<!--
Integration tests, well, test integration between components. The two main 
differences between mock-based unit tests and integration tests are:

1. Integration tests are slower and typically require state management. In our example
we need to care whether MyRealDao persists its state or not, since that makes test
runs unreproducible. If newServiceObject() returns an identical object every time
a foreign key constraint violation might be thrown, so we need to ensure a clean slate ...
-->

Integration testing - clean slate
--
 
    @Before public void ensureCleanSlate() {
      MyDao dao = new MyRealDao(/* config */);
	  dao.deleteAll();
    }
    
<!--
Which we typically do with something like this - before running the test ensure
that we have no previous state. Of course, this can quickly get complicated
if you have multiple related entities so sometimes you simply do a drop/create
of the database. Which is obviously slower.

2. Integration tests works against the implementation, so they may uncover issues
which arise from the difference between contract and implementation.
-->

End-to-end testing
--

    @Test public void login() {
      Client client = new MyBrowserBasedClient();
      AuthResult result = client.login("admin", "admin");
      assertThat(result.isLoggedIn(), is(true));
    }
    
<!--
In the end ( no pun intended ) we have the big, bad, heavy, test-all end-to-end
tests. These typically simulate end-user interaction and exercise the complete
application in a realistic deployment scenario.

There's no point in wondering there these tests are self-contained - they test
everything - from user interface to database access to how components are
wired. They are also the slowest of the bunch and definitely not isolated. And
from my personal experience they're quite flaky and require maintenance.

All of this has subtly build to a recommendation regarding organising tests
for an application...
-->

The testing pyramid ...
--

![Testing pyramid](assets/scaled/testing-pyramid.png)

<!--
Some of you might be familiar with the testing pyramid. The basic idea is to 
have most of your tests be fast tests, basically unit tests. This way
you can get immediate feedback, since a large unit test battery can be quickly
executed by a developer on his local workstation - no setup needed, no flakiness.

Of course, you should test interactions and confirm that a complete setup
works, but you're better served by a large battery of unit tests handling every
variation.

Otherwise you end up with...
-->

... or the testing icecream cone? 
-

![Testing cone](assets/scaled/testing-cone.png)

Testing tools galore
--

* Unit testing
    * Frameworks: JUnit, TestNG
    * Assertions: AssertJ, Truth, fest-assert
    * Mocks: JMock, EasyMock, Mockito, Powermock
* Integration testing: Spring Testing, WireMock, Dumbster, Arquilian
* End-to-end testing: Selenium/WebDriver, REST-assured

<!--
There are plenty of general-purpose of testing libraries, and also some special-purpose 
developed by other projects.

All serve their purpose well and often I find a number of them in Sling-based projects
-->

What about Sling?
======

Sling oddities^W particularities
----

* OSGi
* JCR
* Sling APIs (Java, HTTP)

<!--
But of course, there's always room for more. Sling has some particularities regarding
its technology stack and obviously these shape how the tests look and behave.

First of all we have OSGi as a module and service layer. Typically in tests we care
more about services than modules - the tooling around OSGi bundles is good enough for
us not to care about validating that the imports and exports are right all the time.

Right :-) ?

Then comes JCR. JCR is pretty interesting here because it differs from how applications
are structured - there are no ValueObjects or DataTransferObjects being passed to a Dao,
every object is 'active'.

And last of all, or rather on top of all that, comes Sling. Sling can be seen as both a 
set of central APIs which appear in tests - the ResourceResolver and Adaptable interfaces
come into mind and as various OSGi services that your code interacts with - 
SlingSettingsService, Sling Jobs, discovery etc.
-->

Unit testing with Sling
==

Unit testing
---

	@Test public void test_isRedirectValid_null_empty() {
        TestCase.assertFalse(AuthUtil.isRedirectValid(null, null));
        TestCase.assertFalse(AuthUtil.isRedirectValid(null, ""));
    }
    
Actual code from [AuthUtilTest.java](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/bundles/auth/core/src/test/java/org/apache/sling/auth/core/AuthUtilTest.java)

<!--
A very simple example of a unit test with no dependencies from the Sling code base
-->

Unit testing with ad-hoc mocking
---

	
	@RunWith(JMock.class) public class AuthUtilTest {
      final Mockery context = new JUnit4Mockery();
      final ResourceResolver resolver = context.mock(ResourceResolver.class);
      final HttpServletRequest request = context.mock(HttpServletRequest.class);
      
      @Test public void test_isRedirectValid_invalid_characters() {
			context.checking(new Expectations() { /* code */ })
			TestCase.assertFalse(AuthUtil.isRedirectValid(request, "/illegal/</x"));
      }	        
    }

Actual code also from [AuthUtilTest.java](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/bundles/auth/core/src/test/java/org/apache/sling/auth/core/AuthUtilTest.java)

<!--
This time the tests needs to interact with a ResourceResolver and
HttpServleRequest, so it mocks stuff. I've omitted the actual expectations since
it takes too much space.
-->

Unit testing with Sling mocks
---

	public class ModelAdapterFactoryUtilTest {
	
      @Test
      public void testRequestAttribute() {
        MockSlingHttpServletRequest request = new MockSlingHttpServletRequest();
        request.setAttribute("prop1", "myValue");
        RequestAttributeModel model = request.adaptTo(RequestAttributeModel.class);
        assertNotNull(model);
        assertEquals("myValue", model.getProp1());
      }
    }

Actual code from [ModelAdapterFactoryUtilTest.java](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/testing/mocks/sling-mock/src/test/java/org/apache/sling/testing/mock/sling/context/ModelAdapterFactoryUtilTest.java).

<!--
Another test, this time there is no framework-based mocking, but we're explicitly using a
MockSlingHttpServletRequest ; 
-->
    
Unit testing OSGi code with Sling mocks
---

	public class ExampleTest {
	
	  @Rule
	  public final OsgiContext context = new OsgiContext();
	
	  @Test
	  public void testSomething() {
	
	    // register and activate service
	    MyService service1 = context.registerInjectActivateService(new MyService(),
	        ImmutableMap.<String, Object>of("prop1", "value1"));
	
	    // get service instance
	    OtherService service2 = context.getService(OtherService.class);
	
	  }
	
	}

Picked up from the [OSGi mocks documentation](https://sling.apache.org/documentation/development/osgi-mock.html).

Sling Mocks for unit testing
---

	public class SimpleNoSqlResourceProviderQueryTest {
    
      @Rule public SlingContext context = new SlingContext(ResourceResolverType.JCR_MOCK);
    
      @Before public void setUp() throws Exception {
        context.registerInjectActivateService(new SimpleNoSqlResourceProviderFactory(), ImmutableMap.<String, Object>builder()
                .put(ResourceProvider.ROOTS, "/nosql-simple")
                .build());
        
        // prepare some test data using Sling CRUD API ( not shown here )
    }

      @Test
      public void testFindResources_ValidQuery() {
        Iterator<Resource> result = context.resourceResolver().findResources("all", "simple");
        assertEquals("/nosql-simple", result.next().getPath());
        assertEquals("/nosql-simple/test", result.next().getPath());
        assertEquals("/nosql-simple/test/node1", result.next().getPath());
        assertEquals("/nosql-simple/test/node2", result.next().getPath());
        assertFalse(result.hasNext());
      }
    }
    
Code from [SimpleNoSqlResourceProviderQueryTest](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/contrib/nosql/generic/src/test/java/org/apache/sling/nosql/generic/simple/SimpleNoSqlResourceProviderQueryTest.java)

Sling Mocks for unit testing
---

![Unit testing execution time](assets/original/nosql-unit-test-execution-time.png)

Sling Mocks for unit testing
---

![Unit testing execution time](assets/original/nosql-unit-test-execution-time-magnified.png)


Testing JCR code with Sling mocks
---

    public class FindResourcesTest {

      @Rule public SlingContext context = new SlingContext(ResourceResolverType.JCR_MOCK);

      @Before public void setUp() {
        Resource resource = context.create().resource("test",
                ImmutableMap.<String, Object> builder().put("prop1", "value1")
                        .put("prop2", "value2").build());
		 // snip ...
        MockJcr.setQueryResult(session, Collections.singletonList(node));
    }

    @Test public void testFindResources() {
        Resource resource = context.resourceResolver().getResource("/test");
        Assert.assertNotNull("Resource with name 'test' should be there", resource);

        Iterator<Resource> result = context.resourceResolver().findResources("/test", Query.XPATH);
        Assert.assertTrue("At least one result expected", result.hasNext());
        Assert.assertEquals("/test", result.next().getPath());
        Assert.assertFalse("At most one result expected", result.hasNext());
      }
    }

The humble object pattern with OSGi
---
    public interface RouterAdmin {
	  void doStuff();
	}
	
	public class RouterAdminImpl implements RouterAdmin {
	  // constructor and field elided 
      public void doStuff() {
	    // implementation
	  }
	}
	
	@Component
	@Properties({ @Property(name="url") })
	public class RouterAdminComponent implements RouterAdmin() {
	  private RouterAdmin delegate;

      protected void activate(ComponentContext ctx) throws Exception {
        delegate = new RouterAdminImpl(new URL(requireString(ctx, "url")));
      }
      
      public void doStuff() {
        delegate.doStuff();
      }
	}

See [Humble Object at xUnit patterns](http://xunitpatterns.com/Humble%20Object.html) for more details.

Integration testing with Sling
==

Pax-Exam
--

    public abstract class AbstractJobHandlingTest {
    
      @Inject protected EventAdmin eventAdmin;

      @Inject protected ConfigurationAdmin configAdmin;

      @Inject protected BundleContext bc;
    
      @Configuration public Option[] config() {
        return options(
          frameworkProperty("sling.home").value(new File(...),
          mavenBundle("org.apache.sling", "org.apache.sling.fragment.xml", "1.0.2"),
          mavenBundle("org.apache.sling", "org.apache.sling.fragment.transaction", "1.0.0"),
          mavenBundle("org.apache.sling", "org.apache.sling.fragment.activation", "1.0.2"),
          mavenBundle("org.apache.sling", "org.apache.sling.fragment.ws", "1.0.2"),

          mavenBundle("org.apache.sling", "org.apache.sling.commons.log", "4.0.0"),
          mavenBundle("org.apache.sling", "org.apache.sling.commons.logservice", "1.0.2"),

          mavenBundle("org.slf4j", "slf4j-api", "1.6.4"),
          mavenBundle("org.slf4j", "jcl-over-slf4j", "1.6.4"),
          mavenBundle("org.slf4j", "log4j-over-slf4j", "1.6.4"),

          mavenBundle("commons-io", "commons-io", "1.4"),
          mavenBundle("commons-fileupload", "commons-fileupload", "1.3.1"),
          mavenBundle("commons-collections", "commons-collections", "3.2.1"),
          mavenBundle("commons-codec", "commons-codec", "1.9"),
          mavenBundle("commons-lang", "commons-lang", "2.6"),
          mavenBundle("commons-pool", "commons-pool", "1.6"),

          mavenBundle("org.apache.servicemix.bundles", "org.apache.servicemix.bundles.concurrent", "1.3.4_1"),

    	  // SNIP ...
    	  mavenBundle("org.apache.sling", "org.apache.sling.api", "2.8.0"),
          mavenBundle("org.apache.sling", "org.apache.sling.settings", "1.3.4"),
          mavenBundle("org.apache.sling", "org.apache.sling.resourceresolver", "1.1.6"),
          mavenBundle("org.apache.sling", "org.apache.sling.adapter", "2.1.2"),
		  // SNIP ...
	}
	
Sling mocks with a JCR backend
--

`ResourceResolverType.JCR_MOCK` ?

`JCR_OAK` or `JCR_JACKRABBIT` are also possible

Server-side JUnit tests
--

    @RunWith(SlingAnnotationsTestRunner.class)
    public class OsgiAwareTest {
    
      @TestReference private ConfigurationAdmin configAdmin;
      @TestReference private BundleContext bundleContext;
    
      @Test public void testConfigAdmin() throws Exception {
        assertNotNull( "Expecting ConfigurationAdmin to be injected by Sling test runner", configAdmin);        
      }
    }

Actual code from [OsgiAwareTest](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/testing/samples/sample-tests/src/main/java/org/apache/sling/testing/samples/sampletests/OsgiAwareTest.java)


Scriptable server-side tests
---

	%><sling:defineObjects/><%

    // we don't check for null etc to make the test fail if the service is not available!
    final InfoProvider ip = sling.getService(InfoProvider.class);
    final InstallationState is = ip.getInstallationState();

    String output = "";

    // check 01 : no untransformed resources
    if ( is.getUntransformedResources().size() > 0 ) {
        output += "Untransformed resources: " + is.getUntransformedResources() + "\n";
    }

    // check 02 : no active resources
    if ( is.getActiveResources().size() > 0 ) {
        output += "Active resources: " + is.getActiveResources() + "\n";
    }
    if ( output.length() > 0 ) {
        %><%= output %><%
    } else {
        %>TEST_PASSED<%
    }
	%>

Actual code from [installer-duplicate.jsp](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/launchpad/integration-tests/src/main/resources/scripts/sling-it/installer-duplicate.jsp).

End-to-end testing with Sling
==

HTTP utilities from the Sling testing tools
--

	public class HttpPingTest extends HttpTestBase {
      public void testWebServerRoot() throws Exception {
        // by default, the Launchpad default servlet redirects / to index.html
        final String url = HTTP_BASE_URL + "/";
        final GetMethod get = new GetMethod(url);
        get.setFollowRedirects(false);
        final int status = httpClient.executeMethod(get);
        assertEquals("Status must be 302 for " + url, 302, status);
        final Header h = get.getResponseHeader("Location");
        assertNotNull("Location header must be provided",h);
        assertTrue("Location header must end with index.html", h.getValue().endsWith("index.html"));
      }
    }

Code time
==

Slingshot?
--

![slingshots](assets/scaled/slingshots.jpg)
    
Colophon
--

* Presentation crafted with Eclipse, GNU Make and odpdown
* [Slingshots picture](https://www.flickr.com/photos/81325557@N00/8948946827) by _Anne and Tim_ on Flickr

Final thoughts
--

* Sling provides a lot of specialised testing tools
* Lots of effort recently invested in the unit testing layer
* Experiment and provide feedback