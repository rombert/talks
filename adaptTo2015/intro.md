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
at how do we usually test web-based, data-driven applications.

Then we take these concepts and map them in detail to Apache Sling.

And in the end, we'll take a little time to look at an actual demo of putting the tools
and concepts I've discussed in action.
-->

Speaker.getInstance().interrupt()
===


<!--
Yes, please interrupt me whenever you have question or comment to make.
-->

How do I test my application?
===


<!--
For the purpose of this talk, I will only consider automatic testing of functionality. So no exploratory
testing, no performance testing, no UI-based testing.

I believe that there is great value and lots to be said about those topics, but that Sling-based
applications don't differ that much from other applications in order to justify a separate discussion.


Now that that's done with, let's get on with the concepts about testing
applications in general, to make sure that everyone is on the same page.
-->

Unit testing
--

	@Test public void isNull() { 
	  assertThat( StringUtils.isNull( null ), is(true));
	}
	
<!--
The first form of testing is of course unit testing. Unit tests should be
small, fast and self-contained. These tests validate that a unit - typically
a Java class works well in isolation.

This more or less applies to the 'Utils' family of classes, which don't collaborate
a lot. Which means that in application projects, as opposed to library projects, you
see a small proportion of these tests. In application projects most objects do have
collaborators.
-->
	
Unit testing with mocks
--
 
    @Test public void persist() {
      MyDao dao = mock(MyDao.class);
      MyService service = new MyService(dao);
      service.persist(new ServiceObject()); // must not fail
    }

<!--
And that obviously leads us to a well-known variation of unit test - a unit test
using mocks. Assuming that we have a database-backed application we can test
a service which uses a data access object. 

This trivial example shows how it would look like:

- the DAO is mocked
- the service gets an instance of the dao
- we invoke a method on the service and expect it to not fail

Some points to note here:

1. Mock-based tests work against a contract and not an implementation. We describe what
the collaborator should behave like - in our case that the dao should save the object that
was given to it without throwing an exception. We can even add a verify call to ensure that
the dao was called. But this is all us assuming that we know how the collaborator will behave.

2. Mock-based tests are still small, fast and self-contained. They do tend to
become a bit more verbose than 'plain' unit tests, but are still manageable.
-->

Unit testing with mocks
--
 
    @Test(expected=ServiceException.class) public void persist() {
      MyDao dao = mock(MyDao.class);
      MyService service = new MyService(dao);
      when(dao.persist(anyObject)).thenThrow(new DaoUnavailableException("mocked"));
      service.persist(new ServiceObject());
    }

<!--
Another twist to mocking - since we are in charge of the collaborator's
behaviour, we can simulate various conditions, like a collaborator
throwing an unchecked exception since the dao is - for instance - not able to 
connect to the data source. That can be tedious to arrange if not using mocks.
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

1. Integration tests are slower and typically require state management.

In our example we need to care whether MyRealDao persists its state or not, since that makes test
runs different. If newServiceObject() returns an identical object every time
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

There's no point in wondering these tests are self-contained - they test
everything - from user interface to database access to how components are
wired. They are also the slowest of the bunch and are most realistic measure
of how your application will behave. And from my personal experience they're
quite flaky and require maintenance.

-->

Testing pyramid
--

![Testing pyramid](assets/scaled/testing-pyramid.png)

<!--
So since you have all of these kinds of tests at your disposal, the question is
which should I write. The answer is all of them. Of course, there is no easy way
out. And then, the next question might come, where should I focus most of my effort.
As with all good questions, there is only one answer - it depends.

But I find that the testing pyramid is a good tool to help me decide where I would
focus my efforts. I've hinted that the different kinds of tests have different 
value propositions and different constraints. I like to order them by fidelity,
speed of execution and maintainability. By these criteria, unit tests have the least
fidelity when describing the state of the quality of your sistem, but they are
the most maintainable, and also the absolute fastest to execute.

By extension, for the same the development time, you will be able to run and maintain
much fewer integration or end-to-end tests compared to unit tests. So in terms of
raw efficiency, unit tests win. And as the pyramid suggests, add some integration
tests and some end-to-end tests to ensure that something doesn't slip by and you
have a pretty solid test battery for your application.
-->

... or the testing icecream cone? 
-

![Testing cone](assets/scaled/testing-cone.png)

<!--
Now this is something I've also seen done, with good and bad parts. This is considered
an anti-pattern, but I'm not that dogmatic, and consider it just another approach, in
which the focus is on end-to-end tests, and much less on integration and end-to-end tests.
This usually happens when tests are 'retrofitted' to an existing application, and the
simplest thing to do is write a end-to-end test.

This may work for you, but note that you will have to invest a lot in maintaining these
tests - I know a good number of development teams that have invested a lot in end-to-end
tests and they almost never have a 100% successful testing run, and that's ok.
-->

What about Sling?
======

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
			context.checking(new Expectations() { /* 8 lines of code */ })
			TestCase.assertFalse(AuthUtil.isRedirectValid(request, "/illegal/</x"));
      }	        
    }

Actual code also from [AuthUtilTest.java](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/bundles/auth/core/src/test/java/org/apache/sling/auth/core/AuthUtilTest.java)

<!--
This time the tests needs to interact with a ResourceResolver and
HttpServletRequest, so it mocks stuff. I've omitted the actual expectations since
it takes too much space, but we had a total of 8 lines of expectation code, which 
configure the context path and request attributes
-->

Unit testing with ad-hoc mocking
---

	context.checking(new Expectations() {
	    {
	        allowing(request).getContextPath();
	        will(returnValue("/ctx"));
	        allowing(request).getAttribute(with(any(String.class)));
	        will(returnValue(null));
	    }
	});

<!--
This is that the ad-hoc mocked code actually looks like, not rocket science, but
on the other hand it's a bit verbose.
-->

Unit testing with Sling mocks
---

	public class ModelAdapterFactoryUtilTest {
	
      @Test public void testRequestAttribute() {
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
MockSlingHttpServletRequest. I for one find this style code more natural, and reusing a pre-made
object also eliminates the potential for duplication throught the tests.

A quick note - there is an a rich vocabulary in the testing community regarding test object
semantics - what is a mock, what is a stub, what is a spy, but we generally regard our
test objects as mocks and name them consistently that way.
-->

OSGi
---

![OSGi](assets/scaled/brick-wall.jpg)

<!--
Let me start by saying that I wholeheartedly, no-ifs-or-buts love OSGi. For the last four
years I've used it for Java application I've written, and have not looked back.

What I want to point out though is that while applications benefit greatly from OSGi, 
in my humble opinion, unit tests are better served by different approaches.

Consider the fact that the Sling Launchpad has around 110 bundles ; if you work with
AEM then that number easily doubles. That's not a basis for a unit test. And most of
the bundles directly or indirectly require the Sling repository to be available,
which really adds to startup time.

So while there are option which we can use for OSGi integration testing, at least
for Sling apps, we need a different approach for unit testing.
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

<!--
What this piece of code shows is registering, activating, injecting OSGi services from a bundle, without
having an OSGi framework available.

There are no bundle lists to construct, no private fields injection to satisfy dependencies, it's all done behind
the scenes by Sling Mocks. This opens up a lot of interesting possibilities regarding testing services in isolation,
with test double components ( mocks or otherwise ).
-->

Sling OSGi mocks
--

TODO - a little picture or some concepts about how Sling mocks works

<!--

- note : introduced by sseifert of provision

-->

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
	public class RouterAdminComponent implements RouterAdmin {
	  private RouterAdmin delegate;

      protected void activate(ComponentContext ctx) throws Exception {
        delegate = new RouterAdminImpl(new URL(requireString(ctx, "url")));
      }
      
      public void doStuff() {
        delegate.doStuff();
      }
	}

See [Humble Object at xUnit patterns](http://xunitpatterns.com/Humble%20Object.html) for more details.

<!--

Here's an idea which I've been experimenting with at small scale, based on the Humble Object pattern.

The idea behind this pattern is to extract the logic into a separate easy-to-test component
that is decoupled from its environment. The two classic examples that are cited in favour
of this pattern are asynchronous execution and UI objects. The underlying assumption is that
there is something  in the environment that makes it hard to test the original objects. 

So for instance if we have a UI object - a dialog - which we would like to test and that's hard to
do with classical unit testing since it requires preparing the environment, having a UI available etc,
so we extract all of the logic in a so-called humble object and then we test it in isolation.

My assumption here is that assembling an OSGi environment for a unit test is 'hard', the same
assumption that the OSGi mocks make. But instead of providing a mock OSGi environment, I try 
to decouple the objects from OSGi. Turns out it's not really that hard, since most of what 
we in the Sling world need from OSGi is the SCR and the Metatype service, and we use annotations for that.

As you can see in the code, assuming that we have a RouterAdmin interface, we create two implementations:

- the RouterAdminImpl which does the actual work - our so-called humble object 
- the RouterAdminComponent, which is just the OSGi glue and then delegates all implemented methods
  to the RouterAdminImpl
  
This allows us to test the RouterAdminImpl in isolation. And just as important, objects
depending on the RouterAdmin don't know and don't care if they're talking to the OSGi component
or the humble objects, which means that we can easily substitute something else during unit tests.
-->

JCR
---

![JCR](assets/scaled/oak-tree.jpg)

<!--

Then comes JCR. JCR is pretty interesting here because it has a different
opinion no how persistence should be handled from an API points of view.
You don't usually wirte create ValueObjects or DataTransferObjects
which you pass to a DAO, every object is 'active'.

A JCR node is very much an Active Record in the way Martin Fowler describes
it in 'Patterns of Enterprise Application Architecture' as it is able to
deal with persistence itself.

So the patterns with mocking collaborating DAOs that we saw earlier do not apply.
-->


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

Code taken from [FindResourceTest.java](https://github.com/apache/sling/blob/bc0f839fbcc18ea9d7f75ded51fdbc6ee0e05ba5/testing/mocks/sling-mock/src/test/java/org/apache/sling/testing/mock/sling/jcrmock/resource/FindResourcesTest.java).

<!--
The answer, again, is Sling mocks. Aside from providing a test OSGi-like environment,
Sling mocks provides a number of ResourceResolver implementations out-of-the-box, which 
allows you to work with a ResourceResolver without actually creating a full-fledged
JCR repository, like Jackrabbit or Jackrabbit Oak.

In this test we can see JCR_MOCK as a resource resolver type. This simulates a 
simple JCR implementation, without the bells and whistles like observation, versioning,
ACLs, etc. But you can get away with it is mostly do read-write operations, including queries.

There's also a SLING_MOCK implementation, which you can use if you only work at the
resource level.

This example has some basic usage, creating resources in test, preparing query results
and then validating that results are correct.
--> 

Sling
--

![slingshots](assets/scaled/slingshots.jpg)

<!--
And last of all, or rather on top of all that, comes Sling. Sling can be seen as both a 
set of central APIs which appear in tests - the ResourceResolver and Adaptable interfaces
come into mind and as various OSGi services that your code interacts with - 
SlingSettingsService, Sling Jobs, discovery etc.

Also worth mentioning that for AEM there are also the AEM mocks which provide some
stuff which is tailored for AEM use - they play nicely with the Sling mocks
so look into them if you're testing against AEM.
-->

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

<!--
Not suprisingly, it's also Sling Mocks here. If we can test OSGi and JCR with Sling Mocks, we
can definitely test any Sling code. But on top of that, Sling Mocks also registers some
services out-of-the box:

- AdapterManager
- SlingSettingsService
- MimeTypeService
- ModelAdapterFactory

and also provides other utilities

- ContentLoader for loading data from JSON into the repository
- Mock objects for request handling
- ContentBuilder for programmatic resource creation

Sling Mocks for unit testing
---

![Unit testing execution time](assets/original/nosql-unit-test-execution-time.png)

Sling Mocks for unit testing
---

![Unit testing execution time](assets/original/nosql-unit-test-execution-time-magnified.png)


Integration testing with Sling
==

Pax-Exam
--

    public static Option[] paxConfig() {
        final File thisProjectsBundle = new File(System.getProperty( "bundle.file.name", "BUNDLE_FILE_NOT_SET" ));
        final String launchpadVersion = System.getProperty("sling.launchpad.version", "LAUNCHPAD_VERSION_NOT_SET");
        log.info("Sling launchpad version: {}", launchpadVersion);
        return new DefaultCompositeOption(
                SlingPaxOptions.defaultLaunchpadOptions(launchpadVersion),
                CoreOptions.provision(CoreOptions.bundle(thisProjectsBundle.toURI().toString()))
                
        ).getOptions();
    }
	
Taken from [U.java](https://github.com/apache/sling/blob/bc0f839fbcc18ea9d7f75ded51fdbc6ee0e05ba5/bundles/commons/contentdetection/src/test/java/org/apache/sling/commons/contentdetection/internal/it/U.java)

<!--
Traditionally one of the complicated things with setting up Pax-Exam for Sling was 
putting together the bundle list, which is now made very simple by the SlingPaxOptions
class. This provisions a Sling Launchpad with all the needed bundles so you 
don't have to keep the list up-to-date.
-->

Pax-Exam
--


	@RunWith(PaxExam.class) public class FileNameExtractorImplIT {

      @Inject private FileNameExtractor fileNameExtractor;

      @Test public void testFileNameExtractor(){
        String rawPath = "http://midches.com/images/uploads/default/demo.jpg#anchor?query=test";
        String expectedFileName = "demo.jpg";
        assertEquals(expectedFileName, fileNameExtractor.extract(rawPath));
      }

      @Configuration public Option[] config() {
        return U.paxConfig();
      }
    }

Taken from [FileNameExtractorImplIT.java](https://github.com/apache/sling/blob/bc0f839fbcc18ea9d7f75ded51fdbc6ee0e05ba5/bundles/commons/contentdetection/src/test/java/org/apache/sling/commons/contentdetection/internal/it/FileNameExtractorImplIT.java).

<!--
And then this utility class can be reused in as many tests as you would like.
Simple and concise.
-->
	
Sling mocks with a JCR backend
--

* Unit testing with JCR Mocks
  * JCR_MOCK
  * SLING_MOCK
* Integration testing with JCR Mocks
  * JCR_JACKRABBIT
  * JCR_OAK

<!--
I previously mentioned that there are multiple resource resolver types implemented 
in the Sling mocks. JCR_MOCK and SLING_MOCK are fast, in-memory, implementations
used for unit testing.

But for integration testing you probably want a real JCR repository. And that's
where JCR_JACKRABBIT and JCR_OAK come in. No surprises here, JCR_JACKRABBIT
starts a Jackrabbit 2.x respository while OAK starts an Oak 1.2.x repository.

You can keep the same test cases and switch from one resource resolver type
to another and they should work the same.
-->

Server-side JUnit tests
--

    public class ServerSideInstallerTest {
    
      @Rule public final TeleporterRule teleporter = TeleporterRule.forClass(getClass(), "Launchpad");

      @Before
      public void setup() throws LoginException {
        ip = teleporter.getService(InfoProvider.class);
        is = ip.getInstallationState();
      }
    
      @Test
      public void noUntransformedResources() {
        final List<?> utr = is.getUntransformedResources();
        if(utr.size() > 0) {
            fail("Untransformed resources found: " + utr); 
        }
      }
    }

<!--
The sling testing tools allow you to deploy test code straight into a running Sling instance,
executing them and retrieving the results.

We do this by simply teleporting the code from the test workspace to the running instance.
If you want to know more about this magic, ask Bertrand.

Some key points here:

- we have a TeleporterRule which does most of the work, making sure that the instance is set up
  ( the "Launchpad" argument points to a customiser which handles some of that )
  and a bundle is built on the fly and then deployed to the running instance,
  a test executed and the results retrieved
- the TeleporterRule can access OSGi services, so basically everything is available to you

It's a nice complement to Pax-testing, for the situations where you prefer to provision your
launchpad instance in a different manner.

And also has the coolest name that I know of for a testing tool. Ok, Mockito comes close, but
this is better :-) 
-->


Scriptable server-side tests
---

	// test imports that fail on Java 8, SLING-3405
	
	%><%@page import="java.util.Arrays"%><%
	%><%@page import="java.lang.CharSequence"%><%
	%>TEST_PASSED


Actual code from [installer-duplicate.jsp](https://github.com/apache/sling/blob/e252fc651ab42037af0386bc4cd2b1fc26b13b7b/launchpad/integration-tests/src/main/resources/scripts/sling-it/imports.jsp).

<!--
You can execute scripts as tests. The mechanism is very simple - you only need to output
TEST_PASSED and the test passes. I would recommend using these tests only when you
have core scripting functionality  to test ; otherwise the TeleporterRule is very
much preferred.
-->


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

And finally, if you are looking to write some end-to-end tests, 

Code time
==

Slingshot
--

![Demo](assets/scaled/code.jpg)

<!--
Slingshot is one of the samples from the Sling source tree. It's basically a photo-sharing application,
you can launch it easily into the default launchpad to see some more details. But what I want to do now is to 
show you some Sling Mocks-based tests so you can get the feel of the complete setup.

- pom.xml setup - careful about ordering, note about SNAPSHOTS
- a OSGi-only test
- JCR_MOCK/SLING_MOCK test
- JCR_JACKRABBIT test

-->
    
Colophon
--

* Presentation crafted with Eclipse, GNU Make and [odpdown](https://github.com/thorstenb/odpdown)
* Photo credits
  * [Slingshots](https://www.flickr.com/photos/81325557@N00/8948946827) by _Anne and Tim_ on Flickr
  * [Wall](https://www.flickr.com/photos/bartoszjanusz/6344746584/) by _Bart Lumber_ on Flickr
  * [HTML code](https://www.flickr.com/photos/nikio/3899114449) by _Marjan Krebelj_ on Flickr

Final thoughts
--

* Sling provides a lot of specialised testing tools
* Lots of effort recently invested in the unit testing layer
* Experiment and provide feedback

Resources
--

* Sling mocks
* AEM mocks
* Slingshot

And also the links spread throughout the presentation.