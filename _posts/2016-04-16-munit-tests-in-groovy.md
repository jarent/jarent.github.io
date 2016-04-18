---
layout: post
title: Munit - tests in groovy
---
If I haven't said that earlier, i will do it now - [Munit](https://docs.mulesoft.com/munit/v/1.1.1/) is great! Being able to easly create isolated unit tests, mocking only selected steps in the flow and verifying not only flow output but also value of payload and properties in the middle of flow processing is priceless for developers.

Based on recent releases of library looks like Mulesoft promotes writing Munit tests in XML, thankfully still allowing to do it in [Java](https://docs.mulesoft.com/munit/v/1.1.1/munit-tests-with-java). There are some limitations (not all XML featuures are available), but personally i can live with them. In my opinion tests written that way are more flexible and easier to maintain than the one in XML.
Still i got some frustration with Java language itself. Groovy looks like a good solution for them. I decided to give it a try and write some Munit tests in Groovy.

### 1. Maven set-up

- install groovy eclipse plugin
- create folder src/test/groovy
- add to pom.xml groovy maneve plugin
{% highlight xml %}
<plugin>
    <groupId>org.codehaus.gmavenplus</groupId>
    <artifactId>gmavenplus-plugin</artifactId>
    <version>1.5</version>
    <executions>
        <execution>
            <goals>
                <goal>addTestSources</goal>
                <goal>testCompile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
{% endhighlight %}

- and don't forget to add Munit to pom.xml!
{% highlight xml %}
<properties>
...
	  <munit.version>1.1.1</munit.version>
		<mule.munit.support.version>3.7.0</mule.munit.support.version>
</properties>
...
<dependencies>
...
<dependency>
		   <groupId>com.mulesoft.munit</groupId>
		    <artifactId>mule-munit-support</artifactId>
		    <version>${mule.munit.support.version}</version>
		    <scope>test</scope>
		</dependency>
		<dependency>
		    <groupId>com.mulesoft.munit</groupId>
		    <artifactId>munit-runner</artifactId>
		    <version>${munit.version}</version>
		    <scope>test</scope>
		</dependency>
</dependencies>
{% endhighlight %}

Thanks to those changes Groovy classes with tests are automatically compiled and can be executed as regular Junit tests using standard Eclipse plugin.
![log-sms-eclipse-project](/images/munit-in-groovy/log-sms-eclipse-project.PNG "Log SMS Eclipse project")

### 2. Sample application for tests

It is hard to write tests without application, so let's create one. I can easily testify that I finally was able to follow TDD methodology - I had code for the sample tests before figuring out what tested application would really do!

Finally I decided to create simple application implementing API adding activity history to Salesforce for received SMS.

![log-sms-flow](/images/munit-in-groovy/log-sms-flow.PNG "Log SMS flow")

API is definined in RAML, request contains 3 input parameters in JSON format. There is an integration with Salesforce, so application has to deal with error handling. Logic is simple but requires some test coverage. If you want to play with app remember to update /src/main/app/mule-app.properties with your personal username/password/securityToken.

### 3. Groovy tests
All 4 tests are in LogSMSTest class. Let me show you how groovy can make Mule unit testing more convinient for developer.

#### Easy to read
Being able to avoid not needed parenthesis and semicolons increase test readability.

{% highlight groovy linenos %}
@Test
	public void shouldRejectInvalidRequest() {

		//when											
		MuleMessage result = send '{"phone":"6309999999"}'

		//then
		assert result.getInboundProperty('http.status') == 400
		assert result.getPayloadAsString() == '{ "exception" : {"code": "INVALID_REQUEST", "message": "Bad request" }}'		
	}
{% endhighlight %}

#### Easy to debug
There is no need for extra logging or debugging - failed assertions provide automatically all needed information to identify problem with failed test. You can see below that assertion expected http.status code 401 while flow returned 400.

{% highlight console%}
Assertion failed:

assert result.getInboundProperty('http.status') == 401
       |      |                                 |
       |      400                               false

       org.mule.DefaultMuleMessage
       {
         id=6a8ff2d0-03ed-11e6-ac8a-ecf4bb679c19
         payload=org.glassfish.grizzly.utils.BufferInputStream
         correlationId=<not set>
         correlationGroup=-1
         correlationSeq=-1
         encoding=windows-1252
         exceptionPayload=<not set>

       Message properties:
         INVOCATION scoped properties:
         INBOUND scoped properties:
           connection=close
           content-length=71
           content-type=application/json
           date=Sat, 16 Apr 2016 16:08:11 GMT
           http.reason=
           http.status=400
           mule_encoding=windows-1252
         OUTBOUND scoped properties:
         SESSION scoped properties:
       }
{% endhighlight %}

#### Support for multiline string and template
Imagine that you have complex request that can be used in many test cases that differ only with single value. In Java you will either have to create multiple copies of requests in external resource or import 3rd party template library. Groovy gives it for free and there is no need to read template from external file (althought it is maybe not so good practice after all...)

{% highlight groovy linenos %}
	Template requestTemplate = new groovy.text.SimpleTemplateEngine().createTemplate('''
		 {"LogSMSRequest" :
			{
			"occuredAt": "October 12, 2015 at 08:05PM",
			"whoId": "${whoId}",
			"text":"Hello World!"
			}
			 }
		''')
	//when
	MuleMessage result = send requestTemplate.make([whoId: "00000"]).toString()
	{% endhighlight %}

#### Easy JSON parsing
Assertions with string comparision of whole result works only for simple responses. When complex XML or JSON data is returned, then it has to be parsed in order to create assertions for single field value. Groovy helps with that task with XMLSluper or JSONSlurer classes. Insances of those classes parse string in XML or JSON to map of map and allow accessing values using keys as field names.

For sample response:
{% highlight javascript linenos %}
{ "exception":
	{"code":"INVALID_OPERATION_WITH_EXPIRED_PASSWORD",
	"message": "The users password has expired, you must call SetPassword before attempting any other API operations" }
}   
{% endhighlight %}

assertions could look like this:
{% highlight groovy linenos %}
def json = new JsonSlurper().parseText(result.getPayloadAsString())

json.exception.code == "INVALID_OPERATION_WITH_EXPIRED_PASSWORD"
json.exception.message == "The users password has expired, you must call SetPassword before attempting any other API operations"
{% endhighlight %}

#### Inline maps
Mocking up data for mule application tests requires creating and populating a lot of maps. Not so convenient in pure Java (declare map instance first, then put each key-walue pair in seprarate line..). Groovy helps a lot with that! Take a look on sample inline maps in test below. First one is used together with JSONBuilder to fluently create sample JSON request.
Second time inline map is used to set up mock data returned by Salesforce call. MEL language also treats objects as map, so if you want to mock object to check internal processing there is no need to create real object instance (salesforce api failt in case below), but simply create properly populated map instance.

{% highlight groovy linenos %}
@Test
public void shouldSuccessForValidRequest() {

	//given
	def newTaskId = 'newTaskId'

	//JSON Builder sample
	def requestBuilder = new groovy.json.JsonBuilder()		
	requestBuilder.LogSMSRequest {
			occuredAt "October 12, 2015 at 08:05PM"
			whoId "validId"
			text "Hello World!"
		}						
	//when

	whenMessageProcessor("create").ofNamespace("sfdc").
	withAttributes(['doc:name': 'Create Task']).thenReturn(muleMessageWithPayload(
				//inline map
				[[id:newTaskId,
				  errors:[]
				]]
				))

	MuleMessage result = send requestBuilder.toString()

	//then
	assert result.getInboundProperty('http.status') == 200
	assert result.getPayloadAsString() =~ newTaskId
}
{% endhighlight %}

### 4. Summary
There are still plenty things to improve. For example i wish there is an easy way to combine Munit with [Spock](http://spockframework.github.io/spock/docs/1.0/introduction.html). Unofrtunatelly, both testing frameworks requires test class to extend its framework specific parent class, so both of them can't work together.

After trying writing tests in groovy it is hard to go back to writing tests in Java - at least for me. Give it a try and see how it works for you. The application and tests are available in [github](https://github.com/jarent/munit-groovy).
