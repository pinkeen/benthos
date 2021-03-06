---
title: branch
type: processor
categories: ["Composition"]
beta: true
---

<!--
     THIS FILE IS AUTOGENERATED!

     To make changes please edit the contents of:
     lib/processor/branch.go
-->

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

BETA: This component is experimental and therefore subject to change outside of
major version releases.

The `branch` processor allows you to create a new request message via
a [Bloblang mapping](/docs/guides/bloblang/about), execute a list of processors
on the request messages, and, finally, map the result back into the source
message using another mapping.

```yaml
# Config fields, showing default values
branch:
  request_map: ""
  processors: []
  result_map: ""
```

This is useful for preserving the original message contents when using
processors that would otherwise replace the entire contents.

### Error Handling

If the `request_map` fails the child processors will not be executed.
If the child processors themselves result in an (uncaught) error then the
`result_map` will not be executed. If the `result_map` fails
the message will remain unchanged. Under any of these conditions standard
[error handling methods](/docs/configuration/error_handling) can be used in
order to filter, DLQ or recover the failed messages.

### Conditional Branching

If the root of your request map is set to `deleted()` then the branch
processors are skipped for the given message, this allows you to conditionally
branch messages.

## Fields

### `request_map`

A [Bloblang mapping](/docs/guides/bloblang/about) that describes how to create a request payload suitable for the child processors of this branch.


Type: `string`  
Default: `""`  

```yaml
# Examples

request_map: |-
  root = {
  	"id": this.doc.id,
  	"content": this.doc.body.text
  }

request_map: |-
  root = if this.type == "foo" {
  	this.foo.request
  } else {
  	deleted()
  }
```

### `processors`

A list of processors to apply to mapped requests. When processing message batches the resulting batch must match the size and ordering of the input batch, therefore filtering, grouping should not be performed within these processors.


Type: `array`  
Default: `[]`  

### `result_map`

A [Bloblang mapping](/docs/guides/bloblang/about) that describes how the resulting messages from branched processing should be mapped back into the original payload.


Type: `string`  
Default: `""`  

```yaml
# Examples

result_map: root.foo_result = this

result_map: |-
  root.bar.body = this.body
  root.bar.id = this.user.id

result_map: |-
  root.enrichments.foo = if errored() {
  	throw(error())
  } else {
  	this
  }
```

## Examples

<Tabs defaultValue="HTTP Request" values={[
{ label: 'HTTP Request', value: 'HTTP Request', },
{ label: 'Lambda Function', value: 'Lambda Function', },
{ label: 'Conditional Caching', value: 'Conditional Caching', },
]}>

<TabItem value="HTTP Request">


This example strips the request message into an empty body, grabs an HTTP
payload, and places the result back into the original message at the path
`repo.status`:

```yaml
pipeline:
  processors:
    - branch:
        request_map: 'root = ""'
        processors:
          - http:
              url: https://hub.docker.com/v2/repositories/jeffail/benthos
              verb: GET
        result_map: 'root.repo.status = this'
```

</TabItem>
<TabItem value="Lambda Function">


This example maps a new payload for triggering a lambda function with an ID and
username from the original message, and the result of the lambda is discarded,
meaning the original message is unchanged.

```yaml
pipeline:
  processors:
    - branch:
        request_map: '{"id":this.doc.id,"username":this.user.name}'
        processors:
          - lambda:
              function: trigger_user_update
```

</TabItem>
<TabItem value="Conditional Caching">


This example caches a document by a message ID only when the type of the
document is a foo:

```yaml
pipeline:
  processors:
    - branch:
        request_map: |
          meta id = this.id
          root = if this.type == "foo" {
            this.document
          } else {
            deleted()
          }
        processors:
          - cache:
              resource: TODO
              operator: set
              key: ${! meta("id") }
              value: ${! content() }
```

</TabItem>
</Tabs>


