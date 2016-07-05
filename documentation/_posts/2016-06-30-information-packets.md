---
layout: documentation
title: Information Packets (IP)
categories:
 - documentation
weight: 2
---

- [IP object](#ip-object)
  - [Primary properties](#primary-properties)
    - [Type](#type)
      - [Data](#data)
      - [Open Bracket](#open-bracket)
      - [Close Bracket](#close-bracket)
  - [Optional properties](#optional-properties)
- [Ports API](#ports-api)
  - [Input](#input)
  - [Output](#output)
- [Scope](#scope)

## The IP object <a id="ip-object"></a>

Each packet that a component sends or receives is enveloped in [IP class](https://github.com/noflo/noflo/blob/master/src/lib/IP.coffee). IP objects are normally created and obtained by utilizing Ports API (see section below), this section considers important properties and methods of the object.

### Primary properties <a id="primary-properties"></a>

Any IP object has 2 mandatory properties: `.type` and `.data`.

#### type <a id="type"></a>

<a id="data"></a>
<a id="open-bracket"></a>
<a id="close-bracket"></a>
There are 3 types of objects:
 - `data` - a normal data packet
 - `openBracket` - a packet indicating a beginning of a substream
 - `closeBracket` - a packet indicating an ending of a substream

#### data <a id="data"></a>

This property is the actual value that a packet carries. The default value is `null`. Bracket IPs may also carry non-null data, the semantics of which is user-defined.

### Optional properties <a id="optional-properties"></a>

Arbitrary properties can be set in an IP object. There is a list of such properties recognized by the system and given a special meaning.

 - `groups` - a list of groups for a packet. This is the replacement for the earlier `begingroup` events which are now obsolete.
 - `scope` - see [scope](#scope)
 - `owner` - a reference to a process which currently owns or was the last one to own the packet.
 - `index` - an index of an addressable port on which the IP was received.

## Ports API <a id="ports-api"></a>

A process should use its ports in order to send or receive IPs. We have updated the Ports API so that new components can benefit from using object IPs, while the old components still see them as raw events and raw data.

### Input <a id="input"></a>

A component accepting IP objects as its input should use `process` to handle thme:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    inPorts:
      in:
        datatype: 'string'

  c.process (input, output) ->
    console.log '<' if input.ip.type is 'openBracket'
    console.log '>' if input.ip.type is 'closeBracket'
    console.log input.ip.data
```

For addressable ports the `index` property contains array port index:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    inPorts:
      in:
        datatype: 'string'
        addressable: true

  c.process (input, output) ->
    console.log input.ip.data, "on in[#{input.ip.index}]"
```

### Output <a id="output"></a>

Output ports provide handy methods for sending data enveloped in IP objects.

The most common case is sending data using the `data` method with signature

```coffeescript
data: (data, options = {}, index = null) ->
```

 - `data` - the actual value to be carried
 - `options` - map of any optional IP properties such as `groups`, `scope`, etc.
 - `index` - index for array output ports

There are similar methods to send `openBracket` and `closeBracket` packets:

```coffeescript
openBracket: (data = null, options = {}, index = null) ->
closeBracket: (data = null, options = {}, index = null) ->
```

 - `data` - optional data carried with bracket
 - `options` - map of optional IP properties
 - `index` - index for array output ports

All these 3 methods are *chainable*. It means that a chain of calls will produce a series of packets sent one after another.

The most basic usage example is sending just data value:

```coffeescript
component.outPorts.foo.data 'bar'
```

An example of sending a packet with optional properties:

```coffeescript
component.outPorts.foo.data 'bar',
  scope: 'a1b2c3d4'
  groups: ['tier1']
```

Sending a substream via a chain of calls:

```coffeescript
component.outPorts.foo.openBracket 'accounts'
.data accountRecord1
.data accountRecord2
.closeBracket 'accounts'
```


---------------------
# <a id="scope"></a>Scope

`scope` is a token string that identifies a `scope` of availability for a packet. Packets with the same `scope` are visible across the network(s) within the same context, while packets with different `scope` do not overlap. This is useful e.g. for isolating data that belongs to different requests.

<div class="note">
Every IP has a scope, it is null by default.
</div>

To check the scope of an IP:

```coffeescript
c.process (input, output) ->
  data = input.get 'in'
  console.log data.scope
```

If you need to access the scope of the latest [IP](#ip) being sent in:

```coffeescript
c.process (input, output) ->
  console.log input.scope
```

To set a scope, for example, an express request id:

```coffeescript
express = require 'express'
app = express()
app.get '/eh', (req, res) ->
  attachedInPortSocket.post new noflo.IP 'data', req, scope: req.id
```

When you have to set state and cannot keep things in the [buffer](/process-api/#buffer), assign properties to use scoped indexes, and dont forget to reset the state when required.

```coffeescript
c.example = {}

c.process (input, output) ->
  return unless input.ip.type is 'data'
  data = input.get 'in'
  c.example[data.scope] = data.data
  output.done()

  if true
    delete c.example[data.scope]
```


Concurrency problems in graphs with these graph and these sample components as examples:

<img src="{{ site.baseurl }}/img/concurrency.png" alt="concurrency example graph"></img>

```md
1) a packet (named $1) is sent to start
2) start sends $1 packet to `eh`
3) start sends $1 packet to `repeat`
4) start sends $1 packet to `canada`
```

This is where it gets tricky. Sometimes, `canada` might take longer than `eh` and `repeat`
Sometimes, `eh` might take longer than `canada` and `repeat`. `canada` might use the `db` from previous requests.
So for this run, it might then continue by doing:

```md
5) `repeat` sends a packet to `merge`
6) `eh` sends a packet to `merge`

7) a new packet (named $2) is sent to start
8) start sends $2 packet to `eh`
9) start sends $2 packet to `repeat`
10) start sends $2 packet to `canada`

11) `canada` sends $2 packet to `merge`
12) now `merge` sends out data, but it has $1 from `eh` and `repeat`, but $2 from `canada`!

13) `eh` sends $2 to `merge`
14) `repeat` sends $2 to `merge`
15) now, `merge` sends out data, but it has $2 from `eh` and `repeat`, but $1 from `canada`!
```

When scopes are used, even if the same order of operations happens, it will use the data from the correct request.
