---
layout: post
title: Mule ESB - Real Life Request-Reply
---
Sample batch-processing scenario: 

1. Single HTTP request contains multiple records. 
1. Data in each record must by enhanced with information provided by 3rd party web service, then messages with fixed number of records (exactly 3 - transport channel limitation) delivered to customer. 
1. Processing is successful only if all messages with all records are delivered to customer - partial delivery is not allowed (that's why transaction is spanned on records processing and chunk delivery). 

Initial version of the flow:

![initial-flow](/images/real-life-request-reply/sync-scatter-gather-initial-process.png "Initial flow")

{% highlight xml linenos %}
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
**call-LongRunningTask** invokes http call to 3rd party web service for additional data. It takes always 1 second to complete. For the test input of **10** elements processing time takes **10** seconds and **4** messages are delivered to **finalMessage** vm endpoint.

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

Problems....
====

In production usage there will be millions of records. 
Current performance will not satisfy expectations. In order to improve processing time solution must be switched from single-threaded to multi-threaded processing. Thankfully Mule ESB allows to execute portion of single threaded process with multiple threads using [request-reply scope](http://www.mulesoft.org/documentation/display/current/Request-Reply+Scope). 

So let's refactor inner **foreach** loop of the processing flow to execute **call-LongRunningTask** step in separate thread for each record, then aggregate results and go back to single threaded processing to send output messages - similarly to [scatter-gather pattern](http://www.eaipatterns.com/BroadcastAggregate.html). Results of 20 minutes of work below:

![Request-Reply-Active-Transaction](/images/real-life-request-reply/request-reply-active-transaction1.png "Request-Reply with active transaction")

Main flow with request-reply scope inside which **request** vm outbound endpoint sends records for multi-thread processing, and **reply** vm inbound endpoints receive processing results.
{% highlight xml  %}
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
{% endhighlight %}
![Request-Reply-Active-Transaction](/images/real-life-request-reply/request-reply-active-transaction2.png "Request-Reply Scatter step")

Flow receives records sent to **request** endpoint to split them and pass to **aggr** vm endpoint for further processing. 
{% highlight xml  %}
	<flow name="request-reply-long-task-split" doc:name="long-task-split">
		<vm:inbound-endpoint exchange-pattern="one-way"
			path="request" doc:name="request" />
			
		<collection-splitter enableCorrelation="ALWAYS"
			doc:name="Split records for chunks to Long Running Task" />
		<vm:outbound-endpoint exchange-pattern="one-way"
			path="process" doc:name="aggr" />
	</flow>
{% endhighlight %}
![Request-Reply-Active-Transaction](/images/real-life-request-reply/request-reply-active-transaction3.png "Request-Reply Gather step")

Flow receives record from **aggr** endpoint, invoke **call-LongRunningTask**, waits for all records in **collection-aggragator** stage and finally sends back aggregated results to **reply** endpoint.

{% highlight xml  %}
	<flow name="request-reply-long-task-aggr" doc:name="long-task-aggr">
		<vm:inbound-endpoint exchange-pattern="one-way"
			path="process" doc:name="aggr" />
		<flow-ref name="call-LongRunningTask" doc:name="Call Long Running Task" />
		<collection-aggregator failOnTimeout="true"
			doc:name="Aggregate Results" timeout="120000" />
		<vm:outbound-endpoint exchange-pattern="one-way"
			path="reply" doc:name="reply" />
	</flow>
{% endhighlight %}

Let's run the test and see how it works. Well, it doesn't work at all.

Issue#1: VM endpoint inside request-reply cannot be in transactional scope
====
It turned out to be rather obvious rule. It is highlighted even in official [Mule ESB documentation for Request-Reply Scope](http://www.mulesoft.org/documentation/display/current/Request-Reply+Scope) - warning sign and yellow highlighted note draw attention. However documentation says nothing regarding how to fix it. And it turned out to be quite easy. It is enough to disable transaction for request-reply VM endpoints. It doesn't affect process logic at all - HTTP calls inside **call-LongRunningTaks** are not transactional and **finalMessage** VM outbound endpoint still is within transaction. Explicit definition of proper **vm:transaction** action inside request-reply scope below:
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

Issue #2: Default processing strategies doesn't work
====

Without defined processing strategies each stage in the flow is executed without waiting on results of previous stage. For the simplicity let's start with processing both flows synchronously by setting up processing strategy to **synchronous**:
{% highlight xml %}
<flow name="request-reply-long-task-split" doc:name="long-task-split" processingStrategy="synchronous">
...
<flow name="request-reply-long-task-aggr" doc:name="long-task-aggr" processingStrategy="synchronous">	
{% endhighlight %}

Change didn't help - process still fails, but this time with exception:

{% highlight console %}
ERROR 2014-11-19 23:37:25,351 [connector.VM.mule.default.receiver.07] org.mule.exception.DefaultMessagingExceptionStrategy: 
********************************************************************************
Message               : null
Code                  : MULE_ERROR--1
--------------------------------------------------------------------------------
Exception stack is:
1. null (org.mule.api.store.ObjectAlreadyExistsException)
....
--------------------------------------------------------------------------------
Root Exception stack trace:
org.mule.api.store.ObjectAlreadyExistsException
{% endhighlight %}

After long and not so easy debugging new issue was identified:

Issue #3: If aggregation is needed, avoid nested loops
====
Mule uses internal properties **MULE_CORRELATION_SEQUENCE** and **MULE_CORRELATION_GROUP_SIZE** for tracing split collection elements to enable their aggregation. It turned out that those parameters are overwritten by **foreach** and **splitter** message processors. The solution would be either to store original values of **MULE_CORRELATION_SEQUENCE** and **MULE_CORRELATION_GROUP_SIZE** in outer loop and restore them after nested loop is finished or separate those two iterations. It is much easier to separate two loops.  

![Request-Reply-Active-Transaction](/images/real-life-request-reply/request-reply-final-main-flow.png "Request-Reply Gather step")
{% highlight xml %}
<flow name="request-reply-flow" doc:name="request-reply-flow">
		<http:inbound-endpoint exchange-pattern="request-response"
			host="localhost" port="8181" path="requesReply" doc:name="HTTP" />
		<transactional action="ALWAYS_BEGIN" doc:name="Transactional">
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
			<foreach batchSize="3">											
				<vm:outbound-endpoint path="finalMessage" />
				<logger message="Final Message sent" level="INFO"
				category="MainFlow" doc:name="Logger" />
			</foreach>
			<logger message="All tasks finished: #[payload]" level="INFO"
				category="MainFlow" doc:name="Logger" />
	</transactional>
</flow>
{% endhighlight %}
After that change solution finally works es expected!

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

*All tasks finished* message is at the end of processing, after all records were enhanced and all final messages were sent.

In terms of processing time improvement is significant - right now flows ends within **5 seconds** for **10** records and just **39 seconds** for **1000** records. Nice improvement comparing to original **1000** seconds. 

There are still hidden bottlenecks in the processing and there is a huge gap for error processing, but that will be the subject of next article.

Source code for the flows and sample [munit](https://github.com/mulesoft/munit) tests are available on [github](https://github.com/jarent/real-life-request-reply).

