---
layout: post
title:  "Show promotions based on any custom data with Tridion Smarttarget"
date:   2011-08-26
categories: [development]
tags: [smarttarget]
---
In this blog post I hope to show how easy it is to use your own data within <a href="http://www.sdl.com/en/wcm/products/smarttarget/default.asp">SDL SmartTarget</a>, To execute this example you do need to have a working SmartTarget installation.

Ok, lets get started by imagining we have a web store that sells books and we store information about our visitors, and our Marketeer would like to target certain offers to customers based on which categories they visit the most.
I'm not explaining the customers implementation but i'm assuming a method from the business layer can be called. (the source-code contains a dummy implementation that returns random categories)

Since SDL Tridion 2011 there is an Ambient Data Framework (ADF) on the Content Delivery site. This framework allows sharing of data between applications. You can create so-called cartridges that can retrieve and store data within a request or session scope by specifying input and/or output claims.
The ADF will figure out any dependencies (if cartridge 1 has an input claim, that is an output claim  of cartridge 2, the ADF will make sure cartridge 2 is run first)

SDL SmartTarget comes with a session cartridge, which will provide information like the client's browser, os, ip address etc
SDL Audience Management comes with it's own cartridge which will provide information about the logged in Audience Manager visitor

What we need to do is:

* Get the most looked at category into the Ambient Data Framework
* Configure SmartTarget to use this data when doing a query for promotions
* Add it as a trigger type so we can use it as a trigger in a promotion

#### Writing a cartridge for the Ambient Data Framework
A cartridge consists of a configuration file and one or more ClaimProcessor classes.
The configuration file will contain the input and output claims and which classes will implement the functionality, in our case just one output claim and just one implementing class

<!-- language: xml -->
<pre lang="xml" line="1"><code>&lt;CartridgeDefinition 
        Uri="com:tridion:smarttarget:samples:recommendedbooks" 
        Description="Recommended books cartridge.">
  &lt;ClaimDefinitions>
    &lt;ClaimDefinition 
          Uri="com:tridion:smarttarget:samples:recommendbooks:category" 
          Scope="REQUEST"
          Subject="com:tridion:smarttarget:samples:recommendbooks" 
          Description="The Most lookedat category" />
  &lt;/ClaimDefinitions>
  &lt;ClaimProcessorDefinitions>
    &lt;ClaimProcessorDefinition 
          Uri="com:tridion:smarttarget:samples:recommendbooks" 
          ImplementationClass="com.tridion.smarttarget.samples.recommendbooks.RecommendBooksClaimProcessor" 
          Description="This will put the most looked at category into the claimstore">
      &lt;RequestStart>
        &lt;InputClaims />
        &lt;OutputClaims>
          &lt;ClaimDefinition Uri="com:tridion:smarttarget:samples:recommendbooks:category" />
        &lt;/OutputClaims>
      &lt;/RequestStart>
    &lt;/ClaimProcessorDefinition>
  &lt;/ClaimProcessorDefinitions>
&lt;/CartridgeDefinition>
</code></pre>

and the ClaimProcessor looks like

<!-- language: java -->
<pre lang="java" line="1"><code>public class RecommendBooksClaimProcessor extends AbstractClaimProcessor {

    private final static URI RECOMMENDED_CATEGORY_URI = 
        URI.create("com:tridion:smarttarget:samples:recommendbooks:category");

    @Override
    public void onRequestStart(ClaimStore claimStore) throws AmbientDataException {
        String category = YourBusinessModel.getMostLookedAtCategory();
        claimStore.put(RECOMMENDED_CATEGORY_URI, category, true);
    }
}
</code></pre>

so very simply get your category from your business layer and putting it on the ADF, the boolean at the end, defines whether or not other cartridges can override this value
true meaning it cannot be overwritten

Then the cartridge needs to be added to cd_ambient_conf

<!-- language: xml -->
<pre lang="xml" line="1"><code>&lt;Configuration>
    &lt;Cartridges>
        &lt;Cartridge File="/recommendedbooks_cartridge.xml" />
            &lt;!-- other cartridges removed for brevity -->
    &lt;/Cartridges>
&lt;/Configuration>
</code></pre>

#### Step 2 configuring SmartTarget

Here we configure how the category will be added to the query with a prefix to avoid name conflicts 

<!-- language: xml -->
<pre lang="xml" line="1"><code>&lt;Configuration Version="1.1.0">
    &lt;!-- the rest of the configuration left out for brevity -->
    ..
    &lt;SmartTarget>
        ..
        &lt;AmbientData>
            ..
            &lt;Prefixes>
                .. 
                &lt;com_tridion_smarttarget_samples_recommendedbooks>rb&lt;/com_tridion_smarttarget_samples_recommendedbooks>
            &lt;/Prefixes>
        &lt;/AmbientData>
    &lt;/SmartTarget>
</code></pre>
The name of the element should reflect the base part of the URI for the data stored in the ADF, the last part of the URI together with the prefix will become the name 
so in our case storing "Science Fiction" in
<p>
<pre>com:tridion:smarttarget:samples:recommendbooks:category</pre> 

will end up as 

<pre>rb_category=Science+Fiction</pre> in the query</p>

#### Step 3: configuring the trigger-type

in the config directory of your fredhopper installation there is a trigger-types.xml
in here all the custom trigger-types are configured.

<!-- language: xml -->
<pre lang="xml" line="1"><code>&lt;trigger-types xmlns="http://www.fredhopper.com/schema/knowledge-model/trigger/type/1.0">
    &lt;!-- other trigger-types removed for brevity -->
    ...
    &lt;trigger-type url-param="rb_category" name="Category (most looked at)" basetype="text">
        &lt;list-of-values multiselect="true">
		&lt;value>Fantasy&lt;/value>
		&lt;value>Fiction&lt;/value>
		&lt;value>Romance&lt;/value>
                &lt;value>Science Fiction&lt;/value>
                &lt;value>Thriller&lt;/value>
	&lt;/list-of-values>
    &lt;/trigger-type>
&lt;/trigger-types>
</code></pre>

The multiselect="true" will create a list of checkboxes in the GUI allowing a promotion to trigger on more categories.

if you're list of categories (or any other enumeration) is changing a lot, there is a rest interface that allows you to modify them runtime.
(in Audience Management we do that for instance to propagate changes in Audience management segments to SmartTarget)

The source code for the cartridge is available on [github](https://github.com/gertjana/RecommendBooks), with the sample configurations. 
To run it, you'll need maven and the cd_ambient.jar from the CD Installation in your maven repository
run mvn package, and copy the resulting jar to your website lib where you have the ambient data framework configured.

I hope this has given you some ideas on how to better integrate Smart Target with your business
