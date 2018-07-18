---
author: Nick Beukema
layout: post
---

# Writing Integration Tests

1. Inherit from `IntegrationBaseTest`
1. Include test in non-paralell group
1. Define the test method
1. Wrap entire test in `UITest(() => {})`
1. Available fields on BaseTest (_factory, Browser, etc)
1. Create Page Object
1. Write Page setup and assertions
1. Running the tests
1. Artifacts

## 1. Inherit from `IntegrationBaseTest`

Creating an Integration Test begins with inheriting from `IntegrationBaseTest`. Begin the test in the `MiPlan.Tests.IntegrationTests` namespace:

    namespace MiPlan.Tests.IntegrationTests
    {
      public class MyIntegrationTestSuite : IntegrationBaseTest
      {

      }
    }

*Note: There will be multiple packages to include in this file as well* 

## 2. Include test in non-paralell group

Currently, this setup does not support running tests in parallel, so the `IntegrationBaseTest` file sets up a test collection called "Non-Parallel Collection", add that to the test:


    ...
    [Collection("Non-Parallel Collection")]
    public class MyIntegrationTestSuite : IntegrationBaseTest
    ...

## 3. Define the test method

For this setup, we're using Xunit for tests, so we define a test with the `[Fact]` directive inside of our class:

    ...
    [Fact]
    public void CheckPageHeader()
    {

    }
    ...

## 4. Wrap entire test in `UITest(() => {})`

Inside of the `IntegrationBaseTest`, we have a number of great utilities setup, once being the `UITest` method, which is going to assure we have a brand new server available, a browser to use, and a clear database. It also does cleanup and artifact gathering for us once the test is finished (sucessfully or not).

    ...
    [Fact]
    public void CheckPageHeader()
    {
      UITest(() => {

      })
    }
    ...

## 5. Available fields on BaseTest (_factory, _browser, etc)

We also have a number of fields available to use.

### Browser

This will be the handle to do _anything_ within the browser. Some of the more common uses are visiting urls and grabbing elements to use. Here are some basic usage examples:

    # Visit Urls    
    _browser.Navigate().GoToUrl("http://www.google.com");

    # Grab Element by Id -- <div id="my-element-id"></div>
    _browser.FindElement(By.Id("my-element-id"));

    # Grab Element by Class -- <div class="my-element-class"></div>
    _browser.FindElement(By.ClassName("my-element-class"));

    # Grab Element by CSS Selector -- <div data-test-my-data-attr="1" ></div>
    _browser.FindElement(By.CssSelector("[data-test-my-data-attr='1']"));

    # Get Text from an element
    var myElement = _browser.FindElement(By.ClassName("my-element-class"));
    var myElementText = myElement.Text


To see all of the available methods, visit the [documentation](https://seleniumhq.github.io/selenium/docs/api/dotnet/html/T_OpenQA_Selenium_Remote_RemoteWebDriver.htm)

### Context

In order to interact with the database, the `_context` handle is available in our tests. In order to test that the frontend is displaying data correctly, we need to have that data verifiably in the database first! This is done by Entity Framework calls directly:

    var myThing = new Thing { title = "Thing Title" };
    _context.Thing.Add(myThing);
    _context.Save();

What ever you add to this context will be available when the app is running.

### Factory

In our case, we have a stubbed out secondary service and have it injected through the `_factory`. There are a number of available creation methods within the factory, a few are listed here:

    var student = _factory.CreateStudent();

    # Guids are optional
    var staff = _factory.CreateStaff("Teacher", new Guid("C2FC...D7D"))

See `Factories/GreenBlobFactory.cs` for all available creation methods in the factory.

## 6. Create Page Object

We like to follow the Page Object model for integration tests, where we create a new class to interface with each page. When creating a page class, there is a base `Page` class to inherit from, which will setup our base url and browser to be available throughout. To create a page object, start with the following:

    namespace MiPlan.Tests.IntegrationTests.Pages
    {
      class MyNewPage : Page
      {
        public MyNewPage(FrontendConfig frontendConfig, RemoteWebDriver remoteWebDriver) : base(frontendConfig, remoteWebDriver)
        {

        }
      }
    }

Now when we create this page object, what ever is in the constructor will be executed before we continue. This is where we enter in the page url, seeing as we create a new class for each page.

    ...
    public MyNewPage(FrontendConfig frontendConfig, RemoteWebDriver remoteWebDriver) : base(frontendConfig, remoteWebDriver)
    {
      GoToUrl("/my-new-page");
    }
    ...

`GoToUrl` is a helper that takes in account our base url for the tests, so we don't have to repeat that everywhere.

Now to actually create this page object within our tests, let's jump back into our integration test file and create the `MyNewPage` page object:

    ...
    [Fact]
    public void CheckPageHeader()
    {
      UITest(() => {

        var myPage = new MyNewPage(_frontendConfig, Browser)

      })
    }
    ...

_Note: `_frontendConfig` is provided by the base_

Now that we have our page created, lets jump back into the page object file to create some methods to interface with our page:

    ...
    public MyNewPage(FrontendConfig frontendConfig, RemoteWebDriver remoteWebDriver) : base(frontendConfig, remoteWebDriver)
    {
      GoToUrl("/my-new-page");
    }

    public IWebElement GetHeader()
    {
      return _browser.FindElement(By.ClassName("header-element"));
    }
    ...

We can now use our page object to retrieve the header element on the page:

    ...
    UITest(() => {

      var myPage = new MyNewPage(_frontendConfig, Browser)

      var header = myPage.GetHeader();

    })
    ...

## 7. Assertions

Continueing our example in the section above, we need to check the value of content on the page, we might need to click a few things to get the page in a certain state, but at the end of a test, you always have an assertion:

    ...
    UITest(() => {

      var myPage = new MyNewPage(_frontendConfig, Browser)

      var header = myPage.GetHeader();

      Assert.Equal("My Expected Header Text", header.Text);

    })
    ... 

## 8. Running the tests

To actually run the tests, we use the command `dotnet test`, but this can be used with the `--filter` option to specificy a certain namespace:

    dotnet test --filter "FullyQualifiedName~MiPlan.Tests.IntegrationTests"

The above would run everything within the IntegrationTests namespace. This can be specified down to a certian class of tests, or even one single test.

One important thing to note: Currently if you are not running the tests in Docker, you will have to start up a Selenium server and make sure the frontend is running on the correct port. I would like to include this inside of the framework itself, and a part of the test run eventually.

## 9. Artifacts

At the end of a test run, if there are any failures, there will be artifacts produced to help investigate why the test failed. These will be found in the `tmp` directory within the test project. You will find four different artifacts produced:

  1. A Screenshot of the current failure
  2. Recording of the browser console
  3. Snapshot of the HTML when the failure happened
  4. Stack trace of the system when the failure occured


