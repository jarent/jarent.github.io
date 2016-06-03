---
layout: post
title: Munit - Spock tests as specification
---
I finished [previous post](/munit-tests-in-groovy) complaining that it is hard to combined [Spock](http://spockframework.github.io/spock/docs/1.0/introduction.html) with [Munit](https://docs.mulesoft.com/munit/v/1.2.0/). It is hard, but not impossible. I created library **[spock-munit](https://github.com/jarent/spock-munit/)** as a Spock extension allowing to implement Mule ESB specification using Munit.

### Spock integration with Munit

Library contains single class **[MunitSpecification](https://github.com/jarent/spock-munit/blob/master/src/main/groovy/io/github/jarent/munit/MunitSpecification.groovy)**. It is a composite class extending **[spock.lang.Specification](http://spockframework.github.io/spock/javadoc/1.0/spock/lang/Specification.html)** and creating **muleHelper** instance of **FunctionalMunitSuite** class.

Munit lifecycle methods are called from Spock framework standard methods:

| **Spock**    |    **Munit**     |
| -------------|:----------------:|
| setup()      | __setUpMunit()   |  
| cleanup()    | __restartMunit() |  
| cleanupSpec()| killMunit()  		|  

Thanks to this **muleContext** is created once for suite, refreshed before each test and finally closed after all tests. Standard Munit Java interface methods are exposed in the same way as in **FunctionalMunitSuite**, so classes extending **MunitSpecification** can override:

* getConfigResources()
* haveToDisableInboundEndpoints()
* haveToMockMuleConnectors()

or call:

* runFlow()
* verifyCallOfMessageProcessor()

just as regular Junit tests extending **FunctionaMunitSuite** class.

### Maven set-up

To add spock-munit library to the project:

- clone or download [spock-munit from github](https://github.com/jarent/spock-munit)  
- build it by calling **mvn clean install**
- add munit and groovy dependencies to Mule project (details in [previous post](/munit-tests-in-groovy))
- add spock-munit dependency to Mule project

{% highlight xml %}
<dependency>
		 <groupId>io.github.jarent</groupId>
		 <artifactId>spock-munit</artifactId>
		 <version>0.0.1-SNAPSHOT</version>
		 <scope>test</scope>
		 <exclusions>
			 <exclusion>
				 <groupId>org.codehaus.groovy</groupId>
				 <artifactId>groovy-all</artifactId>
			 </exclusion>
		 </exclusions>
	 </dependency>
 </dependencies>
{% endhighlight %}

Dependency **groovy-all** is excluded, because it is part of Mule ESB server, so it comes together with mule dependencies.

- add separatly spock-core and spock-reports dependencies.

{% highlight xml %}
<dependency>
      <groupId>org.spockframework</groupId>
      <artifactId>spock-core</artifactId>
      <version>1.0-groovy-2.4</version>
      <scope>test</scope>
      <exclusions>
      	<exclusion>
          <groupId>org.codehaus.groovy</groupId>
          <artifactId>groovy-all</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
        <groupId>com.athaydes</groupId>
        <artifactId>spock-reports</artifactId>
        <version>1.2.7</version>
        <scope>test</scope>
        <exclusions>
      	<exclusion>
          <groupId>org.codehaus.groovy</groupId>
          <artifactId>groovy-all</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
{% endhighlight %}

I decided that it is better if Spock dependencies are added separately, because their version depends on Mule ESB version that tests will be executed on. For example Mule ESB 3.8 got groovy version 2.4, so it requires dependency to spock-core-groovy-2.4, while Mule ESB 3.7.3 got groovy version 2.3 and requires dependency to spock-core-groovy-2.3. Short summary in the table below:


| **Mule**&nbsp;&nbsp;&nbsp;     |    **Groovy**&nbsp;&nbsp;     |  **spock-core** |
| -------------|:-----------------:| ---------------:|
|     3.5.x    | 1.8.6 							|  0.7-groovy-1.8 |
|     3.6.x    | 2.3.7 							|  1.0-groovy-2.3 |
|     3.7.x    | 2.3.11 						|  1.0-groovy-2.3 |
|     3.8.x    | 2.4.4  						|  1.0-groovy-2.4 |


Once dependencies are added it is time to write some Spock's specifications for Mule ESB flows!

### Behaviorial Driven Development for Mule

Spock is associated with Behaviorial Driven Development (BDD) movement. BDD testing suggests to write tests as software documentation, focusing on small bits of functionality or sample use cases. The concept is very well explained [here](http://edgibbs.com/spock-intro-a-bdd-testing-framework-in-groovy/).

To show how it can work for Mule ESB I added new test class **[LogSMSSpec](https://github.com/jarent/munit-groovy/blob/master/src/test/groovy/io/github/jarent/LogSMSSpec.groovy)** to [munit-groovy](https://github.com/jarent/munit-groovy) application. I started with rewriting all groovy tests to Spock. All I had to do was:

* remove @Test annotation
* replace **//given, //when, //then** comments with Spock labels (**given:, when:, and:, then:**)
* remove **assert** from conditional expressions in then: block

Tests were already following given-when-then pattern, so it was easy. Final result of refactoring:

{% highlight groovy linenos %}
def "should a Success For Valid Request"() {

		given: "'newTaskId' returned from SalesForce create call"

		def newTaskId = 'newTaskId'
		whenMessageProcessor("create").ofNamespace("sfdc").
		withAttributes(['doc:name': 'Save Task in Salesforce']).thenReturn(muleMessageWithPayload(
					//inline map
					[[id:newTaskId,
					  errors:[]
					]]
					))

		and: "Valid request"
		def requestBuilder = new groovy.json.JsonBuilder()
		requestBuilder.LogSMSRequest {
				occuredAt "October 12, 2015 at 08:05PM"
				whoId "validId"
				text "Hello World!"
			}

		when: "Request is sent"
		MuleMessage result = send requestBuilder.toString()

		then: "Expect 200 http response status code"
		result.getInboundProperty('http.status') == 200

		and: "'newTaskId' returned in the response"
		result.getPayloadAsString() =~ newTaskId
	}
{% endhighlight %}

**Note:** I decided to move calls to **'whenMessageProcess'** to **'given:'**. Setting up mocks belongs rather to setup phase.

### Data Driven testing for Mule

BDD was just a warm up. Now it is time for something really cool! Very often multiple tests follow the same pattern and only differ with values for input parameter or result validation. Below is a sample test showing how data driven testing can be used for testing APIs:

{% highlight groovy linenos %}
@Unroll
def "should #scenario"() {

	expect: "'#responseHttpStatus' http.status response code and #responseMessage as a result"

	MuleMessage result = send request
	result.getInboundProperty('http.status') == responseHttpStatus
	result.getPayloadAsString() == responseMessage

	where: "request is #request"

	scenario 		    |	 request      | responseHttpStatus | responseMessage
	'Reject Invalid Request'    | INVALID_REQUEST |    400		   | '{ "exception" : {"code": "INVALID_REQUEST", "message": "Bad request" }}'
	'Fail When Invalid WhoId'   | VALID_REQUEST   |    500		   | '{ "message": "Failed because of Name ID: id value of incorrect type: 00000" '
}
{% endhighlight %}

In the runtime test method above is unrolled into two separate test methods: **'should Reject Invalid Request'** and **'should Fail When Invalid WhoId'**.

Each method will have own set of values assigned to variables **request, responseHttpStatus** and **responseMessage** according to data table defined in **where:** block.

All instructions from expect: block will be executed automatically changing conditional expressions into groovy assertions.

A lot of 'Groovy' magic is going on with that test. I enlisted some them with links to official Spock documentation:

* [Method unrolling](http://spockframework.github.io/spock/docs/1.0/data_driven_testing.html#_method_unrolling)
* [Data tables](http://spockframework.github.io/spock/docs/1.0/data_driven_testing.html#data-tables)

And that is not all! There are many more way of providing data for test scenarios. The best way to learn them is to go through whole [data drive testing](http://spockframework.github.io/spock/docs/1.0/data_driven_testing.html) section.

### Specification

Many developers like saying 'Source code is the best documentation'. Thanks to Spock unit tests can deliver much better form of software specification, focused more on how solutions is used instead of how it was implemented.

The problem is that going through the details of Spock unit tests source code, altough much easier thanks to their readability, can be hard for non-programmers. There is a great Spock extension to address that issue - **[spock-reports](https://github.com/renatoathaydes/spock-reports)** library which can automatically generates HTML reports with test execution results. Below sample screenshot showing how data driven test method was unrolled in the report:

![spock-reports-sample](/images/spock-munit/spock-reports-sample.png "Sample Spock report")

Full report can be browsed [here](https://rawgit.com/jarent/munit-groovy/master/build/spock-reports/index.html)

**Tip:** There is no need to run maven to refresh Spock reports HTML files - reports are created even when tests are run from Anypoint Studio. By default Spock reports are created in **./build/spock-reports directory**. There is a way to change that directory - details covered in [spock-reports documentation](https://github.com/renatoathaydes/spock-reports).

### Summary

Next time when someone will ask the question 'How this API really works?' I plan to send an email with link to RAML console and attached latest Spock report, generated by my continues integration server as part of latest release of application implementing the API.

No more problems with outdated, inaccurate or incomplete documentation, of course only if I haven't forgot to wrote unit tests covering whole functionality of my application...
