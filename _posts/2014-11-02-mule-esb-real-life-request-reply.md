---
layout: post
title: Mule ESB - Real Life Request-Reply
---
Real-life batch-processing scenario - multiple records in single request, record data must by enhanced with information provided by 3rd party web service, then with fixed sizes (transport channel limitation) delivered to customer. Initial version of the flow below:

![initial-flow](/images/sync-scatter-gather-initial-process.png "Initial flow")

{% highlight xml %}
<flow name="initial-process-flow" doc:name="initial-process-flow">
		<http:inbound-endpoint exchange-pattern="request-response"
			host="localhost" port="8181" path="initialProcess" doc:name="HTTP" />
		<transactional action="ALWAYS_BEGIN" doc:name="Transactional">
			<foreach batchSize="3">
				<foreach>
					<flow-ref name="call-LongRunningTask" doc:name="Flow Reference" />
				</foreach>
				<vm:outbound-endpoint path="finalMessage" />
			</foreach>
			<logger message="All tasks finished: #[payload]" level="INFO"
				category="MainFlow" doc:name="Logger" />
		</transactional>
	</flow>
{% endhighlight %}

Long running task takes 1 second to complete. For the test input of 10 elements becasue of single thread mode total processing time takes 10 seconds and 4 messages are delivered to **finamMessage** vm endpoint.

{% highlight console %}
INFO  2014-11-20 00:07:42,784 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:43,865 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:44,946 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:44,975 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:07:46,903 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:47,996 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:49,096 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:49,110 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:07:50,207 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:51,285 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:52,850 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:52,857 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:07:53,914 [main] LongRunningTask: Task processed
INFO  2014-11-20 00:07:53,924 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:07:53,932 [main] MainFlow: All tasks finished: [Ljava.lang.String;@daf22f0
{% endhighlight %}

In production usage there will be thousands of records. Current performance will not satisfy expectations . In order to improve processing time solution must be switched to multi-threaded mode. Mule ESB allows to refactor portion of single threaded process into some kind of map-reduce by using request-reply scope. So let's refactor inner **foreach** loop, execute **call-LongRunningTask** in separate threads, aggregate results and go back to single threaded processing. Results of 20 minutes of work below:

![Request-Reply-Active-Transaction](/images/request-reply-active-transaction.png "Request-Reply with active transaction")

{% highlight xml %}
<flow name="request-reply-flow" doc:name="request-reply-flow">
		<http:inbound-endpoint exchange-pattern="request-response"
			host="localhost" port="8181" path="requesReply" doc:name="HTTP" />
		<transactional action="ALWAYS_BEGIN" doc:name="Transactional">
			<foreach batchSize="3">
				
					<request-reply doc:name="Request-Reply">
						<vm:outbound-endpoint exchange-pattern="one-way"
							path="request" doc:name="VM" >							
						</vm:outbound-endpoint>
						<vm:inbound-endpoint exchange-pattern="one-way"
							path="reply" doc:name="VM" >							
						</vm:inbound-endpoint>	
					</request-reply>
				
				<vm:outbound-endpoint path="finalMessage" />
			</foreach>
			<logger message="All tasks finished: #[payload]" level="INFO"
				category="MainFlow" doc:name="Logger" />
		</transactional>
	</flow>

	<flow name="request-reply-long-task-split" doc:name="long-task-split">
		<vm:inbound-endpoint exchange-pattern="one-way"
			path="request" doc:name="request" />
			
		<collection-splitter enableCorrelation="ALWAYS"
			doc:name="Split records for chunks to Long Running Task" />
		<vm:outbound-endpoint exchange-pattern="one-way"
			path="process" doc:name="aggr" />
	</flow>

	<flow name="request-reply-long-task-aggr" doc:name="long-task-aggr">
		<vm:inbound-endpoint exchange-pattern="one-way"
			path="process" doc:name="aggr" />
		<flow-ref name="call-LongRunningTask" doc:name="Call Long Running Task" />
		<collection-aggregator failOnTimeout="true"
			doc:name="Aggregate Gryphon Results" timeout="120000" />
		<vm:outbound-endpoint exchange-pattern="one-way"
			path="reply" doc:name="reply" />
	</flow
{% endhighlight %}

Let's run the test and see how it works. Well, it doesn't work at all. Let's see what is going on.

Issue#1: VM endpoint inside request-reply cannot be in transactional scope
====
It is not any black magic. It is highlighted even in official [Mule ESB documentation for Request-Reply Scope](http://www.mulesoft.org/documentation/display/current/Request-Reply+Scope). However documentation says nothing regarding how to fix it. And it turned out to be quite easy - it is enought to disable transaction for request-reply VM endpoints. It doesn't affect process logic at all - HTTP calls are not transactional and final VM outbound endpoint still is within transaction. 
{% highlight xml %}
<request-reply doc:name="Request-Reply">
	<vm:outbound-endpoint exchange-pattern="one-way"
		path="request" doc:name="VM" >
		<vm:transaction action="NOT_SUPPORTED"/>
	</vm:outbound-endpoint>
	<vm:inbound-endpoint exchange-pattern="one-way"
		path="reply" doc:name="VM" >
		<vm:transaction action="NOT_SUPPORTED"/>
	</vm:inbound-endpoint>	
</request-reply>
{% endhighlight %}
After the changes flow finally started to work. But logs don't look right:
{% highlight console %}
INFO  2014-11-19 23:12:09,256 [main] MainFlow: All tasks finished: [Ljava.lang.String;@756aadfc
...
INFO  2014-11-19 23:12:11,045 [call-LongRunningTask.stage1.07] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,091 [call-LongRunningTask.stage1.08] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,082 [call-LongRunningTask.stage1.04] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,111 [call-LongRunningTask.stage1.03] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,112 [call-LongRunningTask.stage1.09] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,123 [call-LongRunningTask.stage1.06] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,123 [call-LongRunningTask.stage1.05] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,136 [call-LongRunningTask.stage1.02] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,140 [call-LongRunningTask.stage1.11] LongRunningTask: Task processed
INFO  2014-11-19 23:12:11,142 [call-LongRunningTask.stage1.10] LongRunningTask: Task processed
{% endhighlight %}
**All tasks finished** before any **Task processed**? It means that aggregation step was executed without waiting for web service responses.
Issue#2: Set up good processing strategies
====
Without processing strategies each stage is executed without waiting on results of previous stage. For the simplicity start with processing both flows synchronously:
{% highlight xml %}
<flow name="request-reply-long-task-split" doc:name="long-task-split" processingStrategy="synchronous">
...
<flow name="request-reply-long-task-aggr" doc:name="long-task-aggr" processingStrategy="synchronous">	
{% endhighlight %}
Changes didn't help - process still fails, but this time with exception"
{% highlight console %}
ERROR 2014-11-19 23:37:25,351 [connector.VM.mule.default.receiver.07] org.mule.exception.DefaultMessagingExceptionStrategy: 
********************************************************************************
Message               : null
Code                  : MULE_ERROR--1
--------------------------------------------------------------------------------
Exception stack is:
1. null (org.mule.api.store.ObjectAlreadyExistsException)
  org.mule.util.store.PartitionedObjectStoreWrapper:48 (http://www.mulesoft.org/docs/site/current3/apidocs/org/mule/api/store/ObjectAlreadyExistsException.html)
--------------------------------------------------------------------------------
Root Exception stack trace:
org.mule.api.store.ObjectAlreadyExistsException
	at org.mule.util.store.PartitionedObjectStoreWrapper.store(PartitionedObjectStoreWrapper.java:48)
	at org.mule.util.store.MonitoredObjectStoreWrapper.store(MonitoredObjectStoreWrapper.java:100)
	at org.mule.routing.requestreply.AbstractAsyncRequestReplyRequester$InternalAsyncReplyMessageProcessor.process(AbstractAsyncRequestReplyRequester.java:324)
    + 3 more (set debug level logging or '-Dmule.verbose.exceptions=true' for everything)
********************************************************************************
{% endhighlight %}

After long and not so easy debugging new issue was identified:

Issue#3: Splitter and foreach uses the same group and sequence id parameters
====
MULE_CORRELATION_SEQUENCE and MULE_CORRELATION_GROUP_SIZE are overwritten between foreach and splitter. The solution would be either to store original values for outer loop and restore them after nested loop is finished or separate those two iterations. It looks better to split two loops. After that looks like solution finally works!

{% highlight console %}
INFO  2014-11-20 00:02:39,940 [connector.VM.mule.default.receiver.04] MainFlow: Split chunk of tasks
INFO  2014-11-20 00:02:39,965 [connector.VM.mule.default.receiver.05] MainFlow: Task started
INFO  2014-11-20 00:02:42,120 [connector.VM.mule.default.receiver.05] LongRunningTask: Task processed
INFO  2014-11-20 00:02:42,195 [connector.VM.mule.default.receiver.07] MainFlow: Task started
INFO  2014-11-20 00:02:42,227 [connector.VM.mule.default.receiver.08] MainFlow: Task started
INFO  2014-11-20 00:02:42,233 [connector.VM.mule.default.receiver.09] MainFlow: Task started
INFO  2014-11-20 00:02:42,236 [connector.VM.mule.default.receiver.05] MainFlow: Task started
INFO  2014-11-20 00:02:42,238 [connector.VM.mule.default.receiver.11] MainFlow: Task started
INFO  2014-11-20 00:02:42,243 [connector.VM.mule.default.receiver.10] MainFlow: Task started
INFO  2014-11-20 00:02:42,245 [connector.VM.mule.default.receiver.04] MainFlow: Task started
INFO  2014-11-20 00:02:42,250 [connector.VM.mule.default.receiver.06] MainFlow: Task started
INFO  2014-11-20 00:02:43,420 [connector.VM.mule.default.receiver.06] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,423 [connector.VM.mule.default.receiver.11] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,466 [connector.VM.mule.default.receiver.08] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,474 [connector.VM.mule.default.receiver.04] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,475 [connector.VM.mule.default.receiver.09] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,481 [connector.VM.mule.default.receiver.10] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,488 [connector.VM.mule.default.receiver.07] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,508 [connector.VM.mule.default.receiver.05] LongRunningTask: Task processed
INFO  2014-11-20 00:02:43,512 [connector.VM.mule.default.receiver.05] MainFlow: Task started
INFO  2014-11-20 00:02:44,596 [connector.VM.mule.default.receiver.05] LongRunningTask: Task processed
INFO  2014-11-20 00:02:44,662 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:02:44,668 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:02:44,674 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:02:44,680 [main] MainFlow: Final Message sent
INFO  2014-11-20 00:02:44,690 [main] MainFlow: All tasks finished: [org.mule.transport.http.ReleasingInputStream@748f93bb, org.mule.transport.http.ReleasingInputStream@7f2d31af, org.mule.transport.http.ReleasingInputStream@2e7157c7, org.mule.transport.http.ReleasingInputStream@2a43e0ac, org.mule.transport.http.ReleasingInputStream@22d9bc14, org.mule.transport.http.ReleasingInputStream@346f41a9, org.mule.transport.http.ReleasingInputStream@1084f78c, org.mule.transport.http.ReleasingInputStream@25f723b0, org.mule.transport.http.ReleasingInputStream@4aa11206, org.mule.transport.http.ReleasingInputStream@40d60f2]
{% endhighlight %}

In terms of speed improvment is also significant - right now flows ends within 5 seconds


Two options for fixing that. Either try to store and 

- error handling (problems with correlation id containg correlation sequence)
