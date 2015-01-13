---
layout: post
title: Mule ESB - Request-Reply error handling
---
[In the previous post]({% post_url 2014-12-09-mule-esb-real-life-request-reply %}) I presented working solution for synchronous batch process with multi-thread processing stage implemented with [request-reply scope](http://www.mulesoft.org/documentation/display/current/Request-Reply+Scope). By 'working' i mean it successfully processed input. But how process will behave, if something goes wrong during processing? Let's test it.

*RequestReplyErrorHandlingTest* is a sample munit test that mocks exception thrown by web service invoked in **call-LongRunningTask**

{% highlight java linenos %}
@Test
	public void shouldInovokeLongRunningProcessConcurrently() throws Exception {
		
		
		whenMessageProcessor("component").ofNamespace("scripting").withAttributes(attribute("name").
					ofNamespace("doc").withValue("Long running activity")).
					thenThrow(new RuntimeException());
		
		 runFlow("request-reply-flow" , testEvent(SAMPLE_INPUT_1_RECORD.split(",")));	
		
	}
{% endhighlight %}

And results are horrible - exception was thrown, process never finished and execution is stuck. Test must be stopped manually. What exactly happen? Diagram below provides good explenation.

![issues-with-no-error-handling](/images/request-reply-error-handling/issues-with-no-error-handling.PNG "Issues with no error handling")

1.	Without reply message **Request-Reply** scope cannot finish and waits indefinitely blocking processing and resources. 
1.	**Aggregate Results** timeout is set up to 10s. But it never triggers
1.	Once **request-reply-long-task-aggr** is stopped even for one chunk, **Aggregate Results** stage will never be completed and final message will never be sent to **reply** vm outbound endpoint

Let's deal with the issues in 'easier first' order.

### Issue #1 - Separate timeout must be set up for request-reply scope
There is a timeout attribute in **Request-Reply** scope definition. After setting up timeout value to 11s. (it should be longer than internal **Aggregate** timeout set up to 10s) **Request-Reply** stage finishes once timeout elapsed with error:

{% highlight console %}
ERROR 2014-12-19 15:33:27,664 [main] org.mule.exception.DefaultMessagingExceptionStrategy: 
********************************************************************************
Message               : Response timed out (11000ms) waiting for message response id "a4c33b50-87c6-11e4-877f-ecf4bb679c19" or this action was interrupted. Failed to route event via endpoint: null. Message payload is of type: String[]
Code                  : MULE_ERROR--2
--------------------------------------------------------------------------------
Exception stack is:
1. Response timed out (11000ms) waiting for message response id "a4c33b50-87c6-11e4-877f-ecf4bb679c19" or this action was interrupted. 
{% endhighlight %}

### Issue #2 - **Aggregate Results** timeout starts counting once first message in the group is aggregated
In the above test it never happenes, because every message fails with exception, so it will never reach aggregate step. Still we can rely on **Request-Reply** timeout - it stops no matter what.

### issue #3 - If even single message in the group failed, **Aggregate Results** stage always finish in timeout
My first idea was to catch exception from remote web service call and pass message to aggregate step. Having second thoughts i realized, that even if processing failed for single chunk, there is no need to wait for the results of other chunks processing. It is better to stop whole processing, so it can be retried or handled in different way. To accomplish that I have added [catch exception strategy](http://www.mulesoft.org/documentation/display/current/Catch+Exception+Strategy) **StopRequestReplyExceptioStrategy** that will set exception message as payload and pass message directly to **reply** vm inbound endpoint.

![issues-with-no-error-handling](/images/request-reply-error-handling/exception-strategy.PNG "Exception Stretegy")

In theory that should allow to finish processing earlier. However the first result was not successful - **reply** vm inbound endpoint from **reqest-reply** rejected message with exception *ObjectAlreadyExistsException* followed by timeout:

{% highlight console %}
ERROR 2015-01-10 13:49:32,462 [connector.VM.mule.default.receiver.06] org.mule.exception.DefaultMessagingExceptionStrategy: 
********************************************************************************
Message               : null
Code                  : MULE_ERROR--1
--------------------------------------------------------------------------------
Exception stack is:
1. null (org.mule.api.store.ObjectAlreadyExistsException)
  org.mule.util.store.PartitionedObjectStoreWrapper:48 (http://www.mulesoft.org/docs/site/current3/apidocs/org/mule/api/store/ObjectAlreadyExistsException.html)

ERROR 2015-01-10 13:49:42,952 [main] org.mule.exception.DefaultMessagingExceptionStrategy: 
********************************************************************************
Message               : Response timed out (11000ms) waiting for message response id "cb044830-9901-11e4-8242-ecf4bb679c19" or this action was interrupted. Failed to route event via endpoint: null. Message payload is of type: String[]
Code                  : MULE_ERROR--2
{% endhighlight %}

It didn't make any sense. How given correlation id could be rejected only to be reported as not delivered 10 seconds later? The explanation is hidden in the code of class **org.mule.routing.requestreply.AbstractAsyncRequestReplyRequester**. 

{% highlight java %}
  protected String getAsyncReplyCorrelationId(MuleEvent event)
    {
        ...
        if (event.getMessage().getCorrelationSequence() > 0)
        {
            correlationId += event.getMessage().getCorrelationSequence();
        ...
        return correlationId;
    }
{% endhighlight %}

Method **getAsyncReplyCorrelationId** is adding correlation sequence to correlation id. And property **MULE_CORRELATION_SEQUENCE** is set on the flow , because **Split** stage is earlier used to divide paylod into chunks. Exception strategy shortcut omits **Aggregate** step, so that additionall property is still there and is automatically added to correlation id. But it can be overwritten in exception strategy

{% highlight xml %}
<set-property propertyName="MULE_CORRELATION_SEQUENCE" value="#['']" doc:name="Property"/>				
{% endhighlight %}

It is rather a hack, not clean solution. Perfect solution should rather remove inbound property MULE_CORRELATION_SEQUENCE, but instead of that inbound property is overwritten by outbound property empty value. This hack causes warning:

{% highlight console %}
WARN  2015-01-11 17:17:48,259 [connector.VM.mule.default.dispatcher.01] org.mule.util.ObjectUtils: Value exists but cannot be converted to int: , returning default value: -1
{% endhighlight %}

Nevertheless it works and request-reply solution got proper error handling - what is proved by the additional error scenario tests available together with final version of the flows on [github](https://github.com/jarent/real-life-request-reply).
