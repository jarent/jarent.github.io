---
layout: post
title: Mule ESB - DataMapper XML input/output	
---
Mule EE [DataMapper]((https://www.mulesoft.com/platform/datamapper)) looks really great on marketing fliers. Who doesn't want to have all-in one graphical mapping solutions? Both ways mappings from/to Excel, CSV, XML, JSON, Pojo with complex transformation logic between.. on top of that streaming support! 
Looks perfect. Unfortunately, in complex XML use-cases scenarios some serious shortcomings affects daily work with that tool. My personal list below:

### 1. Input or output type change requires re-creating whole mapping

You haven't decided if you want to use '''JSON''' or '''Map<k,V>''' on mapping output? Don't event start working with DataMapper yet. You will have to recreate the mappings from the beginning, both for input and output. You can modify mapping elements and attributes, but not mapping input/output type. I understand the need to recreate output if i had to change its type, but why input too? It is very frustrating, especially after you spent some time on carefully configuring fixed-length file layout for 20 or more fields.

### 2. Problems with recursive XSD schema

At work I have to deal with complex and over-bloated XML schemas, with infinite recursion of types. DataMapper cannot handle recursion well as part of input-output metadata creation. Once you loaded your schema and selected root element DataMapper starts creating metadata and soon whole studio is not responding and must be closed, probably due to lack of memory. There is a workaround for that - you can select only specific eleemnts that you plan to use in the mappings with [Filters](https://developer.mulesoft.com/docs/display/current/DataMapper+Concepts#DataMapperConcepts-Filters). But once filtering is enabled, you cannot reach recursive nested elements. I found the answer on mule forum - you have to [set recursion level](http://forum.mulesoft.org/mulesoft/topics/datamapper-cannot-generate-metadata-for-large-xsd#reply_1523589) to the depth that you plan to use.

### 3. Returning output tag attributes as elements

Final one - output XML attributes are returned as elements! Let's say output structure is defined as below:

![xml-output-attributes](/images/datamapper-xml/output-attributes.PNG "output attributes")

Output result from the Datamapper mapping looks unexpected:

{% highlight xml linenos %} 
<?xml version="1.0" encoding="UTF-8"?>
<Test>
  <Node>
    <attr>26</attr>
  </Node>
  <testAttr>13</testAttr>
</Test>
{% endhighlight %}

Each attribute is returned as separate nested element. That's not what I configured. Mystery was solved once I checked '''.grf''' mapping configuration file. DataMapper under the hood uses [Clover ETL](http://www.cloveretl.com/) engine for mapping execution. Meaning of configuration is explained in Clover [XML Writer documentation](http://doc.cloveretl.com/documentation/UserGuide/index.jsp?topic=/com.cloveretl.gui.docs/docs/extxmlwriter.html). It turned out that '''<attr name="mapping">''' value is responsible for the issue:

{% highlight xml linenos %} 
<Node cacheInMemory="true" charset="UTF-8" enabled="enabled" fileURL="dict:outputPayload" guiName="XML WRITER" guiX="900" guiY="20" id="EXT_XML_WRITER0" type="EXT_XML_WRITER">
<attr name="mapping"><![CDATA[<?xml version="1.0" encoding="UTF-8"?>
<Test xmlns:clover="http://www.cloveretl.com/ns/xmlmapping" clover:inPort="0">
  <Node>
    <attr>$0.attr</attr>$0.text
  </Node>
  <testAttr>$0.testAttr</testAttr>
</Test>]]></attr>
<attr name="_data_format"><![CDATA[XML]]></attr>
</Node>
{% endhighlight %}

DataMapper, even though it has fields configured as attribute, creates mapping template as if they were element. After manually converting that mapping template to proper value:

{% highlight xml linenos %} 
<attr name="mapping"><![CDATA[<?xml version="1.0" encoding="UTF-8"?>
<Test xmlns:clover="http://www.cloveretl.com/ns/xmlmapping" clover:inPort="0" testAttr="$0.testAttr">
  <Node attr="$0.attr">
  </Node>
</Test>]]></attr>
{% endhighlight %}

results were finally as expected:

{% highlight xml linenos %} 
<?xml version="1.0" encoding="UTF-8"?>
<Test testAttr="13">
  <Node attr="26"/>
</Test>
{% endhighlight %}

With those manual changes DataMapper graphical editor still worked properly - all previously configured mapping rules were there. But after even slight change in Graphical Editor the configuration file was re-created and all manual changes were gone. Being forced to repeat manual modifications after each DataMapper changes was to much - I gave up.

## Final words..

I still believe in DataMapper. Tool can be very usuefull in flat files processing or extracting data from XML files. Bunch of other powerful futures are enlisted [here](http://blogs.mulesoft.com/7-things-you-didn%E2%80%99t-know-about-datamapper/). But if you want to create complex XMLs better use XSLT transformation or write groovy scripts with '''XMLSlurper'''. DataMapper is not yet there.