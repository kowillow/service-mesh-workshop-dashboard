# Jaeger(예거)를 사용한 분산 추적

[Jaeger][1]는 서비스 메시를 통해 흐르는 요청을 추적할 수 있는 분산 추적 도구입니다. 이는 마이크로서비스 아키텍처의 성능 문제를 디버깅하는 데 매우 유용합니다.

## Jaeger 살펴보기

먼저 Jaeger 사용자 인터페이스를 살펴보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
Jaeger 콘솔을 열기 위해 Jaeger에 대한 엔드포인트를 검색합니다.
</blockquote>

```execute
echo $(oc get route jaeger -n %username%-istio --template='https://{{.spec.host}}')
```

접근 권한을 묻는 메시지가 표시되면 'Allow selected permissions'을 클릭합니다.

> 새 브라우저 탭에서 이 URL로 이동합니다. 제공받은 OpenShift 유저 정보로 로그인합니다.

Once logged in, you should be presented with the Jaeger console:

<img src="images/jaeger-welcome.png" width="1024"><br/>
*Jaeger Welcome*

<br>

You need to create traces to explore how requests flow in your mesh.

<blockquote>
<i class="fa fa-terminal"></i>
Send load to the application user interface:
</blockquote>

```execute
for ((i=1;i<=100;i++)); do curl -s -o /dev/null $GATEWAY_URL; done
```

<br>
Let's inspect the traces for your service.  

On the left bar under **Search**, select `app-ui.%username%` for 'Service' and `boards-%username%.svc.cluster.local` for **Operation**.  

It should look like this:

<img src="images/jaeger-search-boards.png" width="400"><br/>
*Search for Traces to Boards Service*

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Click the 'Find Traces' button and Jaeger should reload with traces to the Boards service.
</blockquote>

<img src="images/jaeger-boards-traces.png" width="1024"><br/>
*Traces to Boards Service*

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Select one of these traces.  
</blockquote>

You'll notice the information includes 'Duration' and 'Total Spans'.  'Duration' indicates the total time it took to send and receive a response for this trace.  'Total Spans' indicates the number of spans; each span represents a unit of work executed for this trace.  In the example below, 'app-ui' took 9.52ms in which 5.14ms was spent on calling the boards service.  The boards service itself took 3.56ms to execute before returning a response.

<img src="images/jaeger-boards-example.png" width="1024"><br/>
*Boards Service Example*

<br>

You can inspect more information about each span by clicking on the span itself.  

<blockquote>
<i class="fa fa-desktop"></i>
Expand the two lowest spans in the tree like this:
</blockquote>

<img src="images/jaeger-boards-expanded.png" width="1024"><br/>
*Boards Service Expanded*

Each span gives you the duration of the span and its start time relative to the total duration.  Under 'Tags', you can see additional information such as the HTTP URL, method, and response.  Finally, you can see the actual IPs of the process that executed this span.  You can verify the IPs match the pods that served this traffic.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
Verify the app-ui pod IP:
</blockquote>

```execute
oc get pods -l deploymentconfig=app-ui -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

<blockquote>
<i class="fa fa-terminal"></i>
Verify the boards pod IP:
</blockquote>

```execute
oc get pods -l deploymentconfig=boards -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

The pod IPs should match to the process IPs in the spans you expanded.

<br>

## Debug User Profile

Let's use what we learned to debug the performance of the user profile service.

<blockquote>
<i class="fa fa-terminal"></i>
Send load to the user profile service:
</blockquote>

```execute
for ((i=1;i<=5;i++)); do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<p><i class="fa fa-info-circle"></i> Wait for this to complete as the profile service is slow.</p>

<br>
Inspect the traces.  
<br>

On the left bar under **Search**, select `app-ui.%username%` for **Service** and `userprofile-%username%.svc.cluster.local` for 'Operation'.  
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
Select 'Find Traces' and Jaeger should reload with traces to the user profile service.
</blockquote>

<img src="images/jaeger-userprofile-traces.png" width="1024"><br/>
*Traces to User Profile Service*

Notice that some of these traces are fast (on the order of ms) and some are slow (about 10s).  

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Select one of the fast traces to start, and expand the lowest span.  
</blockquote>

Your view should look like this:

<img src="images/jaeger-userprofile-fast.png" width="1024"><br/>
*User Profile Fast Service*

In the example above, it took a total of 13.48ms for the trace to complete.  The user profile service itself took 3.5ms to execute and return a response.  You can verify the pod that served this request was the version 1 user profile service.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
Verify the userprofile-v1 pod IP:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile,version=1.0 -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

The pod IP should match to the process IP in the span you expanded.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Now, select one of the slow traces and expand the lowest span.
</blockquote>

Your view should look like this:

<img src="images/jaeger-userprofile-slow.png" width="1024"><br/>
*User Profile Slow Service*

In this view, you can easily see that the total time of the request was spent by the userprofile service itself.  In the example above, it started execution at 5:23ms and took 10 seconds to complete.  You can further verify the pod that served this request was the version 2 user profile service.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
Verify the userprofile-v2 pod IP:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile,version=2.0 -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

The pod IP should match to the process IP in the span you expanded.

<br>

At this point, it is really clear that there is a performance issue directly in the version 2 source.  Although the example was simplistic, distributed tracing is incredibly helpful when you have a complicated network of service calls in your mesh.

<br>


[1]: https://www.jaegertracing.io
