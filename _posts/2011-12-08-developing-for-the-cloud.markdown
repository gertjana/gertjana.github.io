---
layout: post
title:  "Developing for the cloud"
date:   2011-12-08
categories: [development]
tags: [cloud, scala]
---
Lately I'm becoming more and more aware of the possibilities that are out there for developers, both in building, testing and running your applications in the cloud.
So I changed a hobby project i'm working on to be build, tested and deployed in the cloud. and I like to share my experiences here.

Currently i'm using 3 parties to get the picture complete

<ol>
	<li><a title="GitHub" href="http://www.github.com" target="_blank">Github </a>- Online Git repositories with nice collaboration features, if you know it, you'll be using it, if you don't, have a look, you'll never look back</li>
	<li><a title="CloudBees" href="http://www.cloudbees.com/" target="_blank">Cloudbees</a> - Free PaaS with Git Repository and Jenkins CI instance, you could also run your apps here, but no support for Scala, Lift at the time I looked at it</li>
	<li><a title="Cloud Foundry" href="http://www.cloudfoundry.com" target="_blank">Cloudfoundry</a> - Free for now  (Beta) PaaS from VMWare - Support Scala and Lift, which makes it easy for me.</li>
</ol>

For this to work I needed to make a couple of changes to my application that I will explain below.

<h3>Registering a app at cloudfoundry</h3>

Once you have an account and installed the command-line tools you can claim your little application space at the hosted cloudfoundry (or download and run a micro.cloudfoundry instance in your local Vmware system)

``` bash 
vmc target api.cloudfoundry.com
$ vmc login [email]
$ vmc push my-app --path target/war --url my-app.cloudfoundry.com --instances 2 --mem 512k --runtime java
$ vmc create-service mysql my-app-db my-app</code></pre>

What this means is, i'm pushing my war based maven build webapplication to the hosted cloudfoundry as two instances with each 512k memory and runtime java, I'm also binding it to a mysql service called my-app-db.
To see the result of my actions I can type vmc apps

``` bash 
$ vmc apps
+-------------+----+---------+---------------------------+-------------+
| Application | #  | Health  | URLS                      | Services    |
+-------------+----+---------+---------------------------+-------------+
| my-app      | 1  | RUNNING | my-app.cloudfoundry.com   | my-app-db   |
+-------------+----+---------+---------------------------+-------------+
```

The commandline tools also give me options to check logs and crashes, start and stop the application etc.

<h3>Database connectivity</h3>

As cloud applications/instances/services are usually not as easily identifiable with an ip-address, your application needs to find out about services (like a database) in a different manner.
I'm doing this by using the org.cloudfoundry.runtime library to create a mysql Service for me

``` scala 
  object CloudFoundryConnection extends ConnectionManager {
    def newConnection(name: ConnectionIdentifier): Box[Connection] = {
      try {
        import org.cloudfoundry.runtime.env._
        import org.cloudfoundry.runtime.service.relational._
        Full(new MysqlServiceCreator(new CloudEnvironment())
                        .createSingletonService().service.getConnection())

      } catch {
        case e : Exception =&gt; Empty
      }
    }
    def releaseConnection(conn: Connection) {conn.close}
  }
```
  
This object will return a connection if it is running in the cloud, otherwise I'll fall back to the normal database connection of in my case an H2 database which I use for development
so I also changed the DB initialisation part of Lift in its Boot class.
The order of DB discovery is now 1) jndi connection, 2) cloudfoundry 3) connection in Properties file 4) H2 database

``` scala 
if (!DB.jndiJdbcConnAvailable_?) {
  val connection = CloudFoundryConnection
  connection.newConnection(DefaultConnectionIdentifier) match {
    case (Full(connection) =&gt; {
      DB.defineConnectionManager(DefaultConnectionIdentifier, connection)
    }
    case Empty =&gt; {
      val vendor = new StandardDBVendor(
        Props.get("db.class") openOr "org.h2.Driver",
        Props.get("db.url") openOr "jdbc:h2:lift_proto.db;AUTO_SERVER=TRUE",
        Props.get("db.user"),
        Props.get("db.password")
      )
      LiftRules.unloadHooks.append(vendor.closeAllConnections_! _)
      DB.defineConnectionManager(DefaultConnectionIdentifier, vendor)
    }
  }
}
```
    
<h3>Maven integration</h3>

Cloudfoundry also provides a maven plugin that allows most of the functionality to be run as part of a maven goal, which is ideal when you want to deploy from your Jenkins build.
Here are the changes to my pom.xml
Add the repository for the org.cloudfoundry.runtime dependency

``` xml 
&lt;repository>
    &lt;id>springsource-milestones&lt;/id>
    &lt;name>SpringSource Milestones Proxy&lt;/name>
   &lt;url>https://oss.sonatype.org/content/repositories/springsource-milestones&lt;/url>
&lt;/repository>
```

Add the dependency itself

``` xml 
&lt;dependency>
    &lt;groupId>org.cloudfoundry&lt;/groupId>
    &lt;artifactId>cloudfoundry-runtime&lt;/artifactId>
    &lt;version>0.6.1&lt;/version>
&lt;/dependency>
```

For the plugin add the plugin repository

``` xml 
&lt;pluginRepository>
  &lt;id>repository.springframework.maven.milestone&lt;/id>
    &lt;name>Spring Framework Maven Milestone Repository&lt;/name>
    &lt;url>http://maven.springframework.org/milestone&lt;/url>
&lt;/pluginRepository>
```

and the plugin itself

``` xml 
&lt;plugin>
    &lt;groupId>org.cloudfoundry&lt;/groupId>
    &lt;artifactId>cf-maven-plugin&lt;/artifactId>
    &lt;version>1.0.0.M1&lt;/version>
    &lt;configuration>
        &lt;server>mycloudfoundry-instance&lt;/server>
        &lt;target>http://api.cloudfoundry.com&lt;/target>
        &lt;url>medicate.cloudfoundry.com&lt;/url>
        &lt;memory>512&lt;/memory>
        &lt;instances>2&lt;/instances>
    &lt;/configuration>
&lt;/plugin>
```

Username and password information you can specify in the servers section of your .m2/settings.xml file

``` xml 
&lt;server>
    &lt;id>mycloudfoundry-instance&lt;/id>
    &lt;username>email@address.com&lt;/username>
    &lt;password>s3cr3t&lt;/password>
&lt;/server>
```

now you can update your deployed application by a simple

``` bash
$ mvn cf:update
```

<h3> Continous Integration on Cloudbees</h3>

First setup an account at cloudbees.com, create a repository and a Jenkins instance, and make sure cloudbees has your public key to allow git pushes.

Then setup a maven 2/3 build in Jenkins, setting the scm section to watch for changes from your cloudbees git repository.

Make sure your build name doesn't have any spaces in it, cloudbees will choke on that, at least with the scala compiler.

As we need the login information from the <strong>.m2/settings.xml</strong>, you can upload this through webdav to <strong>repository-[username].forge.cloudbees.com/private</strong> directory. 

And then in our jenkins build set <strong>/private/[username]/settings.xml</strong> in the alternative maven settings field (advanced button in the maven section)

I've set the maven goal to <strong>clean scala:doc cf:update</strong> meaning it will compile, test, create scala docs and deploy the application to cloudfoundry.

to finish it up you need to add a remote to cloudbees in your local git repository and push your changes. this then will trigger a build and deploy your app.

``` bash 
git remote add cloudbees ssh://git@git.cloudbees.com/[username]/my-app.git
```

To conclude, besides the 2 i've mentioned, there are a number of solutions out there, that all work in similar ways, but each still has it own requirements, If you want to run Java, Ruby, python, php, node.js applications, with Mysql, postgres, MongoDb, CouchDb, or RabbitMQ, you can get it to work on one or more of these providers.
