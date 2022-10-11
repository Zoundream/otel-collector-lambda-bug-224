  

Steps to reproduce:
- modify `otel-config.yaml` to export to your favorite destination. ideally it's somewhere with some latency to the region you are deploying the lambda to. or even more ideally a destination you control that is designed to respond slowly.
- deploy the lambda with `serverless deploy` and note the HTTP endpoint
- tail the logs of the lambda `serverless logs --function=test-bug-224 -t`
- call the lambda endpoint *once* with curl or via the browser.

What should happen:
- in the logs you should see the results making it to the collector
- the export destination should NOT receive the trace. if it does, it means the export call was too fast and managed to complete before the lambda got frozen.

Once you manage to have a call that did not make it out before the lambda is frozen, wait until the lambda runtime is decommissioned. 

Usually 60-90 seconds seem to be enough, but it varies with no way to know or control, unfortunately.

Once the runtime gets decommissioned, you should see logs similar to this, from the extension:

```json
2022-10-11 11:44:57.154	DEBUG	@opentelemetry/instrumentation-http outgoingRequest on request close()
2022-10-11 11:51:27.557	exporterhelper/queued_retry.go:176	Exporting failed. No more retries left. Dropping data.	{"kind": "exporter", "data_type": "traces", "name": "otlp", "error": "max elapsed time expired rpc error: code = DeadlineExceeded desc = context deadline exceeded", "dropped_items": 1}
go.opentelemetry.io/collector/exporter/exporterhelper.(*queuedRetrySender).onTemporaryFailure
	go.opentelemetry.io/collector@v0.61.0/exporter/exporterhelper/queued_retry.go:176
go.opentelemetry.io/collector/exporter/exporterhelper.(*retrySender).send
	go.opentelemetry.io/collector@v0.61.0/exporter/exporterhelper/queued_retry.go:411
go.opentelemetry.io/collector/exporter/exporterhelper.(*tracesExporterWithObservability).send
	go.opentelemetry.io/collector@v0.61.0/exporter/exporterhelper/traces.go:134
go.opentelemetry.io/collector/exporter/exporterhelper.(*queuedRetrySender).start.func1
	go.opentelemetry.io/collector@v0.61.0/exporter/exporterhelper/queued_retry.go:206
go.opentelemetry.io/collector/exporter/exporterhelper/internal.(*boundedMemoryQueue).StartConsumers.func1
	go.opentelemetry.io/collector@v0.61.0/exporter/exporterhelper/internal/bounded_memory_queue.go:61
2022-10-11 11:51:27.558	zapgrpc/zapgrpc.go:191	[core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {
  "Addr": "api.honeycomb.io:443",
  "ServerName": "api.honeycomb.io:443",
  "Attributes": null,
  "BalancerAttributes": null,
  "Type": 0,
  "Metadata": null
}. Err: connection error: desc = "transport: Error while dialing dial tcp 52.2.107.210:443: i/o timeout"	{"grpc_log": true}
{"level":"debug","msg":"Received ","event :":"{\n\t\"eventType\": \"SHUTDOWN\",\n\t\"deadlineMs\": 1665481889557,\n\t\"requestId\": \"\",\n\t\"invokedFunctionArn\": \"\",\n\t\"tracing\": {\n\t\t\"type\": \"\",\n\t\t\"value\": \"\"\n\t}\n}"}
2022-10-11 11:51:27.563	service/collector.go:196	Received shutdown request
2022-10-11 11:51:27.563	service/service.go:138	Starting shutdown...
2022-10-11 11:51:27.563	pipelines/pipelines.go:118	Stopping receivers...
2022-10-11 11:51:27.563	pipelines/pipelines.go:125	Stopping processors...
2022-10-11 11:51:27.563	pipelines/pipelines.go:132	Stopping exporters...
2022-10-11 11:51:27.563	extensions/extensions.go:56	Stopping extensions...
2022-10-11 11:51:27.563	service/service.go:152	Shutdown complete.
{"level":"debug","msg":"Received SHUTDOWN event"}
{"level":"debug","msg":"Exiting"}
```

