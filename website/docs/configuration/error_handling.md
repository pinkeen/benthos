---
title: Error Handling
---

It's always possible for things to go wrong, be a good captain and plan ahead.

<div style={{textAlign: 'center'}}><img style={{maxWidth: '300px', marginBottom: '40px'}} src="/img/Blobpirate.svg" /></div>

Benthos supports a range of [processors][processors] such as `http` and `lambda` that have the potential to fail if their retry attempts are exhausted. When this happens the data is not dropped but instead continues through the pipeline mostly unchanged, but a metadata flag is added allowing you to handle the errors in a way that suits your needs.

This document outlines common patterns for dealing with errors, such as dropping them, recovering them with more processing, routing them to a dead-letter queue, or any combination thereof.

## Abandon on Failure

It's possible to define a list of processors which should be skipped for messages that failed a previous stage using the [`try` processor][processor.try]:

```yaml
pipeline:
  processors:
    - try:
      - resource: foo
      - resource: bar # Skipped if foo failed
      - resource: baz # Skipped if foo or bar failed
```

## Recover Failed Messages

Failed messages can be fed into their own processor steps with a [`catch` processor][processor.catch]:

```yaml
pipeline:
  processors:
    - resource: foo # Processor that might fail
    - catch:
      - resource: bar # Recover here
```

Once messages finish the catch block they will have their failure flags removed and are treated like regular messages. If this behaviour is not desired then it is possible to simulate a catch block with a [`switch` processor][processor.switch] placed within a [`for_each` processor][processor.for_each]:

```yaml
pipeline:
  processors:
    - resource: foo # Processor that might fail
    - for_each:
      - switch:
        - condition:
            bloblang: errored()
          processors:
            - resource: bar # Recover here
```

## Logging Errors

When an error occurs there will occasionally be useful information stored within the error flag that can be exposed with the interpolation function [`error`][configuration.interpolation]. This allows you to expose the information with processors.

For example, when catching failed processors you can [`log`][processor.log] the messages:

```yaml
pipeline:
  processors:
    - resource: foo # Processor that might fail
    - catch:
      - log:
          message: "Processing failed due to: ${!error()}"
```

Or perhaps augment the message payload with the error message:

```yaml
pipeline:
  processors:
    - resource: foo # Processor that might fail
    - catch:
      - bloblang: |
          root = this
          root.meta.error = error()
```

## Attempt Until Success

It's possible to reattempt a processor for a particular message until it is successful with a [`while`][processor.while] processor:

```yaml
pipeline:
  processors:
    - for_each:
      - while:
          at_least_once: true
          max_loops: 0 # Set this greater than zero to cap the number of attempts
          condition:
            bloblang: errored()
          processors:
            - type: catch # Wipe any previous error
            - resource: foo # Attempt this processor until success
```

This loop will block the pipeline and prevent the blocking message from being acknowledged. It is therefore usually a good idea in practice to use the `max_loops` field to set a limit to the number of attempts to make so that the pipeline can unblock itself without intervention.

## Drop Failed Messages

In order to filter out any failed messages from your pipeline you can use a [`bloblang` processor][processor.bloblang]:

```yaml
pipeline:
  processors:
    - bloblang: root = if errored() { deleted() }
```

This will remove any failed messages from a batch.

## Route to a Dead-Letter Queue

It is possible to route failed messages to different destinations using a [`switch` output][output.switch]:

```yaml
output:
  switch:
    cases:
      - check: errored()
        output:
          resource: foo # Dead letter queue

      - output:
          resource: bar # Everything else
```

[processors]: /docs/components/processors/about
[processor.bloblang]: /docs/components/processors/bloblang
[processor.switch]: /docs/components/processors/switch
[processor.while]: /docs/components/processors/while
[processor.for_each]: /docs/components/processors/for_each
[processor.catch]: /docs/components/processors/catch
[processor.try]: /docs/components/processors/try
[processor.log]: /docs/components/processors/log
[processor.group_by]: /docs/components/processors/group_by
[condition.processor_failed]: /docs/components/conditions/processor_failed
[output.switch]: /docs/components/outputs/switch
[output.broker]: /docs/components/outputs/broker
[configuration.interpolation]: /docs/configuration/interpolation#bloblang-queries
