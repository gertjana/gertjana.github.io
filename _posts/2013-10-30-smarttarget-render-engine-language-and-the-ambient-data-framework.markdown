---
layout: post
title:  "Smarttarget REL (Render Engine language) and the ADF (Ambient Data Framework)"
date:   2013-10-30
categories: [development]
tags: [smarttarget, rel, adf]
---
The next major release of SmartTarget will among other things include support for REL (Render Engine Language).
One of the major use cases we wanted to support is 

Have a website in a technology that isn't covered by our Content Delivery platform and show a webpage by:

* Rendering the page through the Content Delivery Web service
* Render targeted content on that page
* Have promotions that can trigger on (front end) visitor profile information

In the 2013 GA version of SDL Tridion the Content Delivery Web service supports forwarding of claims in the ADF from another web application. This together with the REL support in SmartTarget is what makes it all possible.

I've chosen to use the <a href="http://www.playframework.com">Play framework</a> as my website technology, as this uses my favorite programming language <a href="http://www.scala-lang.org">Scala</a>, can be run in a java based application server and it should be reasonably easy to get the ADF running there.

I'm assuming the Content delivery Webservice is setup and Component Templates and Publication Targets are setup to publish REL content
<h2>1. Setup  the play framework</h2>
Follow <a href="http://www.playframework.com/">these</a> instructions to download/install the play framework (i'm using 2.1 as that is the latest the play2war plugin support)
setup the play2war plugin by following the steps <a href="https://github.com/dlecan/play2-war-plugin">here</a>
set it up for the 2.5 servlet specification
<h2>2. Getting the Ambient Data Framework to run</h2>
<h3>Add the dependency to cd_ambient to Build.scala</h3>
This requires you to have the CD jars deployed to a maven or ivy repository, alternatively you could add the jars in a lib/ folder in the play project.

``` scala 
import sbt._
import Keys._
import com.github.play2war.plugin._
object ApplicationBuild extends Build {
  val appName         = "playTridion"
  val appVersion      = "1.0-SNAPSHOT"

  val appDependencies = Seq(
      "com.tridion" % "cd_ambient" % "7.1.0-SNAPSHOT"
    )

  val main = play.Project(appName, appVersion, appDependencies)
    .settings(Play2WarPlugin.play2WarSettings: _*)
    .settings(
      Play2WarKeys.servletVersion := "2.5",
    )
}
```
<em>* note that I'm using the development snapshots here, replace these with the release version numbers once we've released.</em>

<h3>Configure a web.xml</h3>
To configure the ADF we need to add the Filter information to the web.xml.
The play2war plugin will add any file placed in the war/ folder to the resulting war
So we can add the following web.xml in <strong>/war/WEB-INF/</strong> 

``` xml 
&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;
&lt;web-app version=&quot;2.5&quot;
xmlns=&quot;http://java.sun.com/xml/ns/javaee&quot;
xmlns:xsi=&quot;http://www.w3.org/2001/XMLSchema-instance&quot;
xsi:schemaLocation=&quot;http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd&quot;&gt;
  &lt;filter&gt;
    &lt;filter-name&gt;Ambient Data Framework&lt;/filter-name&gt;
    &lt;filter-class&gt;com.tridion.ambientdata.web.AmbientDataServletFilter&lt;/filter-class&gt;
    &lt;async-supported&gt;true&lt;/async-supported&gt;
  &lt;/filter&gt;
  &lt;filter-mapping&gt;
    &lt;filter-name&gt;Ambient Data Framework&lt;/filter-name&gt;
    &lt;url-pattern&gt;/*&lt;/url-pattern&gt;
  &lt;/filter-mapping&gt;
  &lt;listener&gt;
    &lt;listener-class&gt;play.core.server.servlet25.Play2Servlet&lt;/listener-class&gt;
  &lt;/listener&gt;
  &lt;servlet&gt;
    &lt;servlet-name&gt;play&lt;/servlet-name&gt;
    &lt;servlet-class&gt;play.core.server.servlet25.Play2Servlet&lt;/servlet-class&gt;
  &lt;/servlet&gt;
  &lt;servlet-mapping&gt;
    &lt;servlet-name&gt;play&lt;/servlet-name&gt;
    &lt;url-pattern&gt;/&lt;/url-pattern&gt;
  &lt;/servlet-mapping&gt;
&lt;/web-app&gt;
```
And we need to add the cd_ambient_conf.xml to <strong>/war/WEB-INF/classes</strong>

<h3>Test if it works</h3>

Play uses the MVC pattern by setting up routes which are URL Patterns that have corresponding Controller methods which will return rendered views.
Let's show a list of claims in a page by setting up a route and controller plus view for it in play
in <strong>/conf/routes</strong> add the following line

``` bash
GET     /claims                     controllers.Application.claims()
```
this means for every request to /claims the Application.claims() method will be called
The claims() method in the controller looks like this:

``` scala -->
  def claims = {
    val cs = Option(AmbientDataContext.getCurrentClaimStore)
    Action {
      val claimsFuture = Future(cs match {
        case Some(cs) => {
          cs.getAll().map(c => (c._1.toString -> c._2))
        }
        case None => {
          Logger.error("no claim store found")
          mutable.Map[String, AnyRef]()
        }
      })
      Async {
        claimsFuture.map { claims => Ok(views.html.claims(claims)) }
      }
    }
  }
```

First thing i'm doing is getting the claimstore, as Play is setup to be asynchronous from the bottom up, and the ADF stores the claimstore in a ThreadLocal variable during the request, once i'm in the Action{} part i'm actually in another thread. 

I'm wrapping it in an Option class, which is scala's way to avoid having to deal with null values.
Then I'm just converting it from a Java HashMap&lt;URI, Object&gt; into a Scala mutable.Map[String, AnyRef] and passing it onto the view:

``` html
@(claims: scala.collection.mutable.Map[String, AnyRef])
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
    &lt;title&gt;Claims&lt;/title&gt;
    &lt;link rel="stylesheet" media="screen" href="@routes.Assets.at("stylesheets/main.css")" /&gt;
    &lt;link rel="shortcut icon" type="image/png" href="@routes.Assets.at("images/favicon.png")" /&gt;
    &lt;script src="@routes.Assets.at("javascripts/jquery-1.9.0.min.js")" type="text/javascript"&gt;&lt;/script&gt;
&lt;/head&gt;
&lt;body&gt;
    &lt;table&gt;
        &lt;tr&gt;
            &lt;th&gt;Claim URI&lt;/th&gt;
            &lt;th&gt;Value&lt;/th&gt;
        &lt;/tr&gt;
        @claims.map { claim =&gt;
            &lt;tr&gt;
                &lt;td&gt;@claim._1&lt;/td&gt;
                &lt;td&gt;@claim._2&lt;/td&gt;
            &lt;/tr&gt;
        }
&lt;/table&gt;
&lt;/body&gt;
&lt;/html&gt;
```

Calling the url will show a table with the name/values of all claims in the claimstore.

If you have any cartridges for the ADF, simple add them as appDependencies to your <strong>/project/Build.scala</strong> and in your <strong>cd_ambient_conf.xml</strong> and the claims should appear in this page.

``` scala
  val appDependencies = Seq(
    "com.tridion" % "cd_ambient" % "7.1.0-SNAPSHOT",
    "com.tridion.smarttarget.ambientdata" % "session_cartridge" % "2.1.0-SNAPSHOT"
  )
```
<h2>3. Forwarding claims</h2>

<h3>Configuring webservice</h3>

To forward claims from your webapplication to the Content Delivery Web service you need to serialize them and pass them on with your request as a cookie, in your Web service you need to whitelist the IP and claims you accept and set the name of the cookie to use in your cd_ambient_conf.xml on the webservice tier.

``` scala
&lt;Configuration&gt;
  ...
  &lt;Security&gt;
    ...
    &lt;WhiteList&gt;
      &lt;IPAddresses&gt;
      &lt;Ip&gt;10.100.0.0-10.100.255.255&lt;/Ip&gt;
      &lt;Ip&gt;127.0.0.1&lt;/Ip&gt;
    &lt;/IPAddresses&gt;
    &lt;/WhiteList&gt;
    &lt;GloballyAcceptedClaims&gt;
      &lt;Claim Uri=&quot;taf:claim:ambientdata:sessioncartridge:useragent:os&quot; /&gt;
      &lt;Claim Uri=&quot;taf:claim:ambientdata:sessioncartridge:useragent:os:version&quot; /&gt;
      &lt;Claim Uri=&quot;taf:claim:ambientdata:sessioncartridge:useragent:browser&quot; /&gt;
    &lt;/GloballyAcceptedClaims&gt;
  &lt;/Security&gt;
  &lt;Cookies&gt;
    ...
    &lt;Cookie Type=&quot;ADF&quot; Name=&quot;TAFContext&quot; /&gt;
  &lt;/Cookies&gt;
  ...
&lt;/Configuration&gt;
```

<h3>Adding the Claims as a cookie</h3>

CD has a helper method to serialize the claims, it will convert the claims to JSON and then base64 encode them.
It will also split them up to avoid reaching the maximum length a cookie can have. the highlighted line is the Content Delivery Class that does the serializing.

``` scala
  private def getClaimsAsCookies():Seq[(String, String)] = {
    Option(AmbientDataContext.getCurrentClaimStore) match {
      case Some(cs) => {
        val claimCookieSerializer = new ClaimCookieSerializer("TAFContext")
        val serializedClaims = claimCookieSerializer.serializeClaims(cs.getAll())
        
        serializedClaims.map(cookie => {
          ("Cookie", cookie.getName + "=\"" + new String(cookie.getValue) + "\"")
        })
      }
      case None => {
        Logger.error("No claimstore, nothing to serialize")
        Nil
      }
    }
  }
```

This method will return a Sequence of cookie headers that can be added to the request to the webservice

<h2>3. Requesting a Page from Tridion</h2>

For simplicity I will just create a route that get's a page based on it's tcmuri
also for simplicity I will not create a client model.

``` scala
GET     /page/:id                   controllers.Application.page(id)
```

Then the controller method will look like this:

``` scala
  private lazy val baseUrl =  Play.current.configuration.getString("tcdweb").get
  private val pagePattern = "/Pages(PublicationId=%s,ItemId=%s)/PageContent"
  def page(pageId:String) = {
    val headers = getClaimsAsCookies()

    Action {
      val pageUri = new TCMURI(pageId)
      val url = baseUrl + pagePattern.format(pageUri.getPublicationId, pageUri.getItemId)

      val responseFuture = WS.url(url).withHeaders(headers: _*).get()
      Async {
        responseFuture.map {
          resp => {
            Ok(views.html.page(getContentFromXml(resp.body, "Content")))
          }
        }
      }
    }
  }

  private def getContentFromXml(xmlText: String, tagName:String):String = {
    val xml = scala.xml.XML.loadString(xmlText)
    val content = xml \\ tagName filter ( z => z.namespace == "http://schemas.microsoft.com/ado/2007/08/dataservices")
    content.text
  }
```

The webservice url is taken from the configuration, and again the getClaimsAsCookies() method is outside the Action to avoid the issue of the claimstore being in another thread.

As I'm rendering a complete page, my view template is very simple the @Html() is there to avoid html entities being escaped.

``` html
@(content: String)

@Html(content)
```

<h2>4. SmartTarget REL Tag example</h2>

Just for reference this is a simple Page Template that uses SmartTarget to output some promotions in a header and sidebar region

``` html
&lt;!DOCTYPE html PUBLIC &quot;-//W3C//DTD XHTML 1.0 Strict//EN&quot; &quot;http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd&quot;&gt;
&lt;html xmlns=&quot;http://www.w3.org/1999/xhtml&quot; xmlns:tridion=&quot;http://www.tridion.com/ContentManager/5.0&quot;&gt;
  &lt;head&gt;
    &lt;!-- TemplateParam name=&quot;Page&quot; type=&quot;boolean&quot; value=&quot;true&quot; --&gt;
    &lt;title&gt;@@Page.Title@@&lt;/title&gt;
  &lt;/head&gt;
  &lt;body style=&quot;font-family: Verdana; margin:0; padding:0; border:0;&quot;&gt;
    &lt;h1&gt;Smart Target Promotions&lt;/h1&gt;
    &lt;tcdl:query publication=&quot;@@Publication.ID@@&quot;&gt;
      &lt;h1&gt;Header&lt;/h1&gt;
      &lt;div&gt;
        &lt;tcdl:promotions region='Header' maxItems='5'&gt;
          &lt;tcdl:itemTemplate&gt;
            &lt;tcdl:promotionalItems&gt;
              &lt;tcdl:itemTemplate&gt;
                &lt;tcdl:ComponentPresentation ComponentURI='##componentUri##' TemplateURI='##templateUri##' Type='Dynamic'/&gt;
              &lt;/tcdl:itemTemplate&gt;
            &lt;/tcdl:promotionalItems&gt;
          &lt;/tcdl:itemTemplate&gt;
          &lt;tcdl:fallbackContent&gt;...
          &lt;/tcdl:fallbackContent&gt;
        &lt;/tcdl:promotions&gt;
      &lt;/div&gt;
      &lt;h1&gt;Sidebar&lt;/h1&gt;
      &lt;div&gt;
        &lt;tcdl:promotions region='Sidebar' maxItems='5'&gt;
          &lt;tcdl:itemTemplate&gt;
            &lt;tcdl:promotionalItems&gt;
              &lt;tcdl:itemTemplate&gt;
                &lt;tcdl:ComponentPresentation ComponentURI='##componentUri##' TemplateURI='##templateUri##' Type='Dynamic'/&gt;
              &lt;/tcdl:itemTemplate&gt;
            &lt;/tcdl:promotionalItems&gt;
          &lt;/tcdl:itemTemplate&gt;
          &lt;tcdl:fallbackContent&gt;...
          &lt;/tcdl:fallbackContent&gt;
        &lt;/tcdl:promotions&gt;
      &lt;/div&gt;
    &lt;/tcdl:query&gt;
  &lt;/body&gt;
&lt;/html&gt;
```

<h2>Food for thought</h2>

To make my life a little easier, I was cheating a little bit by using a technology that is compatible with Java so I could use Content Deliveries method to serialise claims and of course the Ambient Data Framework itself.

If for instance you would do this in a technology that is even further removed from Java, say Javascript, Ruby or even PHP, you'll need to serialise the claims yourself. 

Luckily the format is very well described on <a href="http://sdllivecontent.sdl.com">SDL Live Content</a>, a search for "JSON Cookie format" will give you all the information you need.

If Javascript is your technology of choice you might want to take look at the <a href="https://github.com/puntero/CT4T/wiki">CT4T Project</a> by Angel Puntero and some other MVP's and the related blog articles by Will Price on <a href="http://www.tridiondeveloper.com/client-side-templating-with-json-odata-and-angular">Tridion Developer</a> and  Alexander Klock on <a href="http://codedweapon.com/2013/10/client-templating-for-tridion-part-1/">Coded Weapon</a>

You can already do everything except showing targeted content with the SDL Tridion 2013 Release, to able to use REL for your SmartTarget content, you have to be patient a little while longer.

<strong>[Update] As of the end of april 2014 SmartTarget 2014 is released making al this possible.</strong>

The source code for this application can be found in my <a href="https://github.com/gertjana/playtridion">github repository</a>

Oh yeah besides being a Developer in SDL I'm also the Technical Product Owner for the SmartTarget product line within <a href="http://www.sdl.com/">SDL</a>