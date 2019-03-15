---
layout: post
title:  "Unittesting custom tags with selenium webdriver"
date:   2011-11-29
categories: [development]
tags: [selenium, java, testing]
---
In the company I work in we have some complex tags that are not easily unit-testable because there are some dependencies between them, the best option here should be to re-factor them, unfortunately as this would break backward compatibility, we have a process of deprecation to go through until we can actually do that.

But code with hardly any tests on them always makes me feel uncomfortable. so I decided to do some (almost integration) tests but running as part of the normal unit test suite
I'm doing this by placing the tags on a jsp, running this on a embedded Jetty Server and using <a href="http://seleniumhq.org/docs/03_webdriver.html">Selenium Webdriver</a> to test the resulting html.

First the base class, which takes care of starting/stopping the embedded Jetty Server and setting up the web driver.

``` java 
public abstract class BaseTest {
    private static int port = 9595;
    private static Server server = new Server(port);

    protected static String baseUrl = "http://localhost:" + port + "/";
    protected static WebDriver driver;

    @BeforeClass
    public static void setup() throws Exception {

        //configure jetty as an exploded war
        URL webAppUrl = BaseTest.class.getClassLoader().getResource("/");
        WebAppContext wac = new WebAppContext();
        wac.setContextPath("/");
        wac.setWar(webAppUrl.toExternalForm());

        server.setHandler(wac);
        server.setStopAtShutdown(true); //makes sure the sever is stopped even if the @AfterClass is never reached
        server.start();

        //HtmlUnit drive doesn't give a popup, the rest of the drivers do
        driver = new HtmlUnitDriver();
    }

    @AfterClass
    public static void teardown() throws Exception {
        driver.close();
        server.stop();
    }
```

In my project I have setup a resources directory that contains a WEB-INF/ with a web.xml containing my custom tag definitions and my tld file as you would with a normal web project.

Webdriver comes with a set of Drivers for different browsers (Firefox, chrome, IE, iPhone, Android, etc) for testing browser compatibility, I don't care about that, the tags I'm testing do not output any html/css that might require this.

The HtmlUnit driver is internal and doesn't popup a browser window so ideal for my purpose.

Writing a test now becomes very easy:

In this case I have a custom tag that when invoked simply outputs "Hello World"

my test jsp:

``` html 
&lt;%@ page language="java" contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"  isELIgnored="false" %>
&lt;%@ taglib uri="/customtaglib" prefix="custom" %>
&lt;html>
  &lt;head>
    &lt;title>Hello World&lt;/title>
  &lt;/head>        
  &lt;body>
    &lt;p><custom:helloworld />&lt;/p>
  &lt;/body>
&lt;/html>
```

The Test class:

``` java
public class HelloWorldTest extends BaseTest {

    @Test
    public void testHelloWorldTag() {
        driver.get(baseUrl + "helloworld.jsp");
        Assert.assertEquals("Hello World", driver.getTitle());

        WebElement element = driver.findElement(By.cssSelector("body p"));
        Assert.assertEquals("Hello World", element.getText());
    }
```

Testing the resulting html is made very easy, by having the WebElement findElement() and List<WebElement> findElements() methods, and then setting predicates with the By class. for instance By.cssSelector(), By.tagName() etc.

It is also possible to test click throughs and form submits, by calling click() or submit() on WebElements.

Added bonus is that although the tag code is running in the embedded jetty server and not in the unit tests itself it's coverage is measured by our code coverage tools

One thing to look out for is for the embedded Jetty to work with JSP's you need to add jetty's version of the jsp spec to your project not the javax.servlet.* ones.
my maven dependencies look likes this:

``` xml
    &lt;dependencies>
        &lt;dependency>
            &lt;groupId>org.seleniumhq.selenium&lt;/groupId>
            &lt;artifactId>selenium-java&lt;/artifactId>
            &lt;version>2.12.0&lt;/version>
        &lt;/dependency>
        &lt;dependency>
            &lt;groupId>org.mortbay.jetty&lt;/groupId>
            &lt;artifactId>jetty&lt;/artifactId>
            &lt;version>6.1.22&lt;/version>
        &lt;/dependency>
        &lt;dependency>
            &lt;groupId>org.mortbay.jetty&lt;/groupId>
            &lt;artifactId>jsp-api-2.1&lt;/artifactId>
            &lt;version>6.1.14&lt;/version>
        &lt;/dependency>
         &lt;dependency>
            &lt;groupId>org.mortbay.jetty&lt;/groupId>
            &lt;artifactId>jsp-2.1&lt;/artifactId>
            &lt;version>6.1.14&lt;/version>
        &lt;/dependency>
        &lt;dependency>
             &lt;groupId>junit&lt;/groupId>
             &lt;artifactId>junit&lt;/artifactId>
             &lt;version>4.7&lt;/version>
             &lt;scope>test&lt;/scope>
        &lt;/dependency>
    </dependencies>
```



