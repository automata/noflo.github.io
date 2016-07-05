---
layout: documentation
title: Process API
categories:
 - documentation
weight: 6
---

The main idea behind process api is having all [port events](/documentation/information-packets/#type) come into one place, and all of the [output](#sending)s sent out from the same place.

First of all let's see what a component defined using Process API looks like:

```coffeescript
exports.getComponent = ->
  new noflo.Component
    description: "Forward received data"
    inPorts:
      in:
        datatype: 'all'
        description: 'An input port'
    outPorts:
      out:
        datatype: 'all'
        description: 'An output port'
    process: (input, output) ->
      return unless input.has 'in'
      data = input.getData 'in'
      output.send data
      output.done()
```

Note the `process: (input, output) ->`, that's the main method on
Process API and gives its name. The `input` is a reference to all
incoming IPs to the component and `output` is a reference to all
outgoing IPs from the component.

The way the process api works is the async process function gets called for each event. If `done` does not get called, the process function will getting called, and the IPs that are passed to it keep getting appended to the buffer.

<div class="note">
The <code>component.process</code> returns an instance of component, so you don't have to
return it on your component definition.
</div>

-----------------------------
### Index
- [Component States](#component-states)
  - [Preconditions](#preconditions)
  - [Processing](#processing)
    - [Getting](#getting)
    - [Sending](#sending)
  - [Done](#done)
  - [Stream Helpers](#stream-helpers)
    - [hasStream](#has-stream)
    - [getStream](#get-stream)
  - [Data Stream Helpers](#data-stream-helpers)
    - [hasDataStream](#has-data-stream)
    - [getDataStream](#get-data-stream)
    - [Flat Stream](#flat-stream)
- [Firing Patterns](#firing-patterns)
  - [Full Stream](#full-stream)
  - [Per Packet](#per-packet)
- [Brackets](#brackets)
  - [BracketForwarding](#bracket-forwarding)
- [Ordering](#ordering)
  - [Auto Ordering](#auto-ordering)
  - [Ordered](#ordered)
- [Buffer](#buffer)

----------

# <a id="component-states"></a> Component States

1) Components _start_
2) Components process _preconditions_ are checked
3) Once Components preconditions are passed, they begin _processing_.
4) Finally, once Components finish processing and call `output.done`, they are _done_.

----------------------------------------

## <a id="preconditions"></a> Preconditions

Given that `input` represents all the input received by a component, it is
common to check for some preconditions before _firing_ a component.

A common operation is to check if `input` has a packet arriving at some port:

```coffeescript
input.has 'portname'
```

Using `has` with multiple arguments will check if all those inports have packets:

```coffeescript
input.has 'portname', 'secondportname'
```

An important detail to note about `input.has` is that it will check for any packet type
(i.e. `openBracket`, `data`, `closeBracket`). In this context, `input.has` can be given
a callback argument that
can check any IP attributes to answer if the IP matches the custom requirements or not.
For example, if someone wants only `data` IPs, it is possible to filter them
using the callback:

```coffeescript
hasHoldData = input.has 'hold', (ip) -> ip.type is 'data'
```

----------------------------------------

## <a name="processing"></a>Processing

Once a component has passed the preconditions, it begins processing. Processing is where you get the data, possibly perform operations on data, and can send IPs out before calling `output.done`.

## <a name="getting"></a>Receiving

### Get <a name="get"></a>
When someone wants to receive data in a component, `input.get` will get the first
IP from the [buffer](#buffer) of that port.

For example, if the incoming packet flow on an `in` inPort is:
```md
1) openBracket
2) data
3) closeBracket
```

If `input.get 'in'` is called, first it will receive the `openBracket`
(remember that `openBracket` is also an IP).
If called again, it will receive `data`, and then again after that, would
receive `closeBracket`.

Since `input.get` removes an IP from the buffer each time, you can repeatedly
call it until you have what you need. For example, if you want to collect all
`data` IPs:

```coffeescript
data = input.get 'in'
until data.type is 'data'
  data = data.get 'in'
```

### GetData <a name="get-data"></a>
The `input.getData portname` is a shortcut for `input.get(portname).data`.

If the port name is not passed in as an argument, it will try to retrieve from
the `in` inport. Meaning, `input.getData()` is the same as `input.getData 'in'`.

<div class="note">
when you <pre>input.get|getData</pre> from a <pre>control</pre> port, it does not reset the <pre>control</pre> ports buffer because the data is meant to persist until new data is sent to that <pre>control</pre> port. <pre>control</pre> ports also only accept <pre>data</pre> ips. If it is sent bracket <pre>IP</pre>s, they will be dropped silently.
</div>

As said, `input.getData` will accept port name(s) as the parameter.
Passing in one port will give the data:

```coffeescript
data = input.getData portname
```

Passing in multiple port names will give an array of the data (using [destructuring](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)):

```coffeescript
[canada, eh] = input.getData 'canada', 'eh'
```

<div class="note">
using <pre>input.get</pre> and <pre>input.getData</pre> will remove the item retreived using it from the buffer.
</div>

## <a name="sending"></a>Sending

Considere `output` as the representation of all data sent by the component.
If you're taking something and `send`ing multiple [IPs](/information-packets),
you should make them a `Stream`, meaning it should be wrapped with an
`openBracket` and an `closeBracket`:

```coffeescript
output.send new noflo.IP 'openBracket'

for data in ['eh', 'igloo']
  output.send out: data

output.send new noflo.IP 'closeBracket'
output.done()
```

If you're just sending one packet out of one port, it is usually best to use
`output.sendDone` which is a shortcut for `output.send` + `output.done`:

```coffeescript
output.sendDone out: 'data'
```

Which is the same as:

```coffeescript
output.send out: 'data'
output.done()
```

It is also possible to send to multiple ports at a time using an _output map_
like:

```coffeescript
output.sendDone
  first: 'foo'
  second: 'bar'
```

The following is an example of a component that sends to multiplor ports at a
time or error if there is an error:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    description: 'Take in data on 1 inport, repeat that data out a port
      and send other data out of another'
    inPorts:
      eh:
        datatype: 'string'
    outPorts:
      canadian:
        datatype: 'all'
      igloo:
        datatype: 'all'
      error:
        datatype: 'object'

  c.forwardBrackets =
    eh: ['canadian', 'igloo']

  c.process (input, output) ->
    return unless input.has 'eh'

    eh = input.getData 'eh'

    unless eh?
      # will send to `error` outport
      output.sendDone new Error('inPort `eh` did not have any real data!')
      return

    output.sendDone
      canadian: '+1'
      igloo: eh
```

Sending [IP](/documentation/information-packets)s

There are generally 2 cases when you’d want to send `noflo.IP` explicitly:

- When sending `openBracket` or `closeBracket`
- When specifying IP metadata, such as `scope`, `index` or application-specific metadata.

<!-- I didn't understand why do I need this? What's the bennefit of sending IP
instead of some data packet? -->

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    description: 'get data on an inport, send out as a specifically scoped IP'
    inPorts:
      in:
        datatype: 'all'
    outPorts:
      out:
        datatype: 'all'

  c.process (input, output) ->
    return unless input.has 'in'

    data = input.getData 'in'
    output.sendDone out: new noflo.IP('data', data, scope: 'example-scope')
```

To see more usage of sending, including using streams,
check out [writing your own projects guide component, FindEhs](/projects/find-ehs).

----------------------------------------
## <a name="done">Done</a>

When you are done processing your data, call `output.done()`
(or `output.sendDone` if it makes sense for how you're using it.)

<div class="note">
An <code>Error</code> can be send to <code>output.sendDone</code> or
<code>output.done</code> which
will send the <code>Error</code> to the <code>error</code> port.
If there is not an <code>error</code>
port defined, it will propogate back up. The same happens if you just
throw an <code>Error</code>.

<pre><code>output.sendDone new Error('we have a problem')</code></pre>

In the future, it may emit a proccesserror.
</div>

----------------------------------------

# <a name="Brackets"></a>Brackets
Brackets are [Information Packets](/documentation/information-packets/) used to
group things. Read more about their different types in
[Information Packets Types](/documentation/information-packets/#type).

## <a name="BracketForwarding"></a>BracketForwarding
[animation]()

<div class="note">
Brackets are automatically forwarded from <code>in</code> inport to
outports <code>out</code> and <code>error</code> (if those ports exist).
</div>

Bracket forwarding is a way to pass on brackets so that you don't have to
deal with brackets coming from that inport in the process function.

If an inport receives an `openBracket`, `data`, and `closeBracket`
and you are using the `bracketForwarding: true` option, you can get the
`data`, process it and send [IPs](/information-packets) out. What you
send out will be wrapped in the `openBracket` and `closeBracket`.

For example, given some [IPs](/information-packets) coming into an `in` port:

```md
1) openBracket ('name')
2) data
3) closeBracket ('name')
```

and the component handling the [IPs](/information-packets):

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    inPorts:
      eh:
        datatype: 'all'
    outPorts:
      canada:
        datatype: 'object'
    forwardBrackets:
      eh: ['canada']
    process: (input, output) ->
      return unless input.has 'eh'
      output.send canada: 'one'
      output.send canada: 'two'
      output.done()
```

a socket listening to the `canada` outport would receive:

```md
1) openBracket ('name')
2) 'one'
2) 'two'
3) closeBracket ('name')
```

When sending brackets as a group, the `openBracket` and `closeBracket`
should contain the same data.

Control ports are not wrapped with brackets, they only deal with data.

A more advanced example using sub-streams (should be avoided if possible because they add unnecessary complexity):

```md
1) openBracket, '$outtermost'

2) openBracket, '$sub1'
3) data, 'eh'
4) closeBracket, '$sub2'

5) openBracket, '$sub2'
6) data, 'canada'
7) data, 'igloo'
8) closeBracket, '$sub2'

9) openBracket, '$outtermost'
```

and using the following component to handle the stream:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    inPorts:
      in:
        datatype: 'all'
    outPorts:
      out:
        datatype: 'object'
    process: (input, output) ->
      return unless input.has 'in'
      data = input.get 'in'
      until data.type is 'data'
        data = input.get 'in'

      output.send data + ' eh!'
      output.done()
```

the output stream would be:

```md
1) openBracket, '$outtermost'

2) openBracket, '$sub1'
3) data, 'eh eh!'
4) closeBracket, '$sub2'

5) openBracket, '$sub2'
6) data, 'canada eh!'
7) data, 'igloo eh!'
8) closeBracket, '$sub2'

9) openBracket, '$outtermost'
```

@TODO:
What's happening here is forwardBrackets is

An example of bracket forwarding can be found in
[Loading Components inline](/documentation/testing/#loading-components-inline).

-----------------------------------------------------

# <a id="stream-helpers"></a>Stream Helpers

## hasStream <a id="has-stream"></a>
Will check if an input port has the full stream. A full stream is all
the IPs surrounded by open/close brackets, as [in this example](#full-stream).

```coffeescript
input.hasStream portname
```

It can also take multiple port names as arguments:

```coffeescript
input.hasStream 'eh', 'canada'
```

## getStream <a id="get-stream"></a>
Will get the [full stream](#full-stream) and then reset the
[buffer](#buffer) state for that port.

```coffeescript
stream = input.getStream portname
```

It can also take multiple port names as arguments:

```coffeescript
[eh, canada] = input.getStream 'eh', 'canada'
```

For an example of how to use streams,
see [DetermineEmotion](https://github.com/aretecode/canadianness/blob/master/components/DetermineEmotion.coffee) component.

--------
# <a id="data-stream-helpers"></a>Data Stream Helpers

Data Stream helpers are available so a component can receive a full stream,
yet only have to deal with only the
[data](/documentation/information-packets/#data) IPs and let
[bracketForwarding](#bracket-forwarding) option deal with the
[brackets](/documentation/information-packets/#open-bracket).
The data stream helpers are mainly used for ports that receive
[Flat Streams](#flat-streams).

## hasDataStream <a id="has-data-stream"></a>

hasDataStream checks similarily to [hasStream](#has-stream), however, when using `data: true` on the port and allowing [bracketForwarding](#bracket-forwarding) to do things behind the scenes, it has a different way of checking.

[hasStream](#has-stream) checks if every openBracket has a closeBracket. But when forwardBrackets is enabled for a port, IPs that are not data are removed from the buffer, so there has to be a separate value to track the IPs that come in that are not data. Additionally, when using forwardBrackets, process function is triggered _before_ the last `closeBracket`, so using `data: true` changes it to be triggered _after_.

<div class="note">
<code>hasDataStream</code> will only work if the port <code>data</code> property is <code>true</code>.
</div>

## getDataStream <a id="get-data-stream"></a>

Often when using streams, all that is required is getting the `data` from the stream.

Using only `getStream` that might look like this:

```coffeescript
c.process (input, output) ->
  return unless input.hasStream 'in'
  stream = input.getStream 'in'
    .filter (ip) -> ip.type is 'data'
    .map (ip) -> ip.data
  console.log stream
```

This can be achieving much easier using the `getDataStream` helper:

```coffeescript
c.process (input, output) ->
  return unless input.hasStream 'in'
  stream = input.getDataStream 'in'
  console.log stream
```

## Flat Streams <a id="flat-streams"></a>
[Data Streams](#data-stream-helpers) using the
[hasDataStream](#has-data-stream) are usually only beneficial using
[flat streams](#flat-streams). A flat stream has only one pair of
open/close brackets, in other words there are no nested no sub-streams,
like in the following example:

```md
1) openBracket, $outtermost
2) data, 'eh'
3) data, 'canada'
4) data, 'igloo'
5) closeBracket, $outtermost
```

The following is a _non flat stream_ because the data has sub-streams.
When input comes in, sometimes packets are wrapped in sub-streams.
For example (after the comma is the data in the packet):

```md
1) openBracket, $outtermost

  2) openBracket, $sub-stream-a
  3) data, 'eh'
  4) closeBracket, $sub-stream-a

  5) openBracket, $sub-stream-b
  6) data, 'canada'
  7) data, 'igloo'
  8) closeBracket, $sub-stream-b

9) closeBracket, $outtermost
```

If using this example sub-stream with Data Stream helpers using this example component:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    description: 'takes in data, and appends "eh" to the end of each string'
    inPorts:
      in:
        datatype: 'string'
        data: true
    outPorts:
      out:
        datatype: 'string'
    process: (input, output) ->
      return unless input.hasDataStream 'in'
      stream = input.getDataStream 'in'
      for data in stream
        output.send data + ' eh!'
      output.done()
```

The resulting output stream would have lost the sub-stream brackets:

```md
1) openBracket, $outtermost
2) data, 'eh eh!'
3) data, 'canada eh!'
4) data, 'igloo eh!'
5) closeBracket, $outtermost
```

See [noflo-packets/Compact](https://raw.githubusercontent.com/aretecode/noflo-packets/549b67f9e500a958e0283f9d4b8308b43aac66d7/components/Compact.coffee) for an example using [hasDataStream](#has-data-stream) and [getDataStream](#get-data-stream).

-----------------------------------------------------
# <a id="firing-patterns"></a> Firing Patterns

There are two standard firing patterns

<!-- OK, I'm a bit confused... so there're two patterns:

  1. full stream: < data, data, data, ... >
  2. per packet: < data, data, data > (ignoring all open/close brackets)

  Is that right? Or do we have another pattern:

  3. per packet with no open/close brackets: data, data, data, ...
-->

This is data being sent to a port in order:
```
1) openBracket
2) data
3) data
4) closeBracket
```

sometimes, there is no `openBracket` or `closeBracket`, and there is only `data`:

```
1) data
```


## <a id="full-stream"></a> Full Stream


```md
# get everything from here...
1) openBracket
2) data
3) data
4) closeBracket
# ...to here
```

or if there is no wrapping brackets:

```md
# get everything from here...
1) data
# ...to here
```

example implementation:

```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    icon: 'gear'
    inPorts:
      eh:
        datatype: 'all'
        required: true
    outPorts:
      canada:
        datatype: 'object'
        required: true
    process: (input, output) ->
      return unless input.hasStream 'in'
      stream = input.getStream 'in'
      # ...do stuff with the stream...
      output.sendDone canada: stream
```


## <a id="per-packet"></a>Per Packet
```md
1) openBracket # don't get this
2) data # get only this
3) data # and then only this
4) closeBracket # don't get this
```

example:
```coffeescript
exports.getComponent = ->
  c = new noflo.Component
    icon: 'gear'
    inPorts:
      eh:
        datatype: 'all'
        required: true
    outPorts:
      canada:
        datatype: 'object'
        required: true
    c.forwardBrackets =
      eh: ['canada']
    process: (input, output) ->
      return unless input.has 'eh', (ip) -> ip.type is 'data'
      data = input.getData 'eh'
      until data.type is 'data'
        data = data.get 'eh'
      # ...do stuff with the data...
      output.send canada: data
```



----------------------------------------
# Ordering <a id="ordering"></a>

## Ordered <a id="ordered"></a>
The `ordered` component option that makes the component maintain the order between `input` and `output` regardless of streams. (_default is `false`_)

<div class="note">
By default, component outport is ordered when using <a href="#sending">output.send</a>.
</div>

For example, a synchronous `KnexDbSelect` component that outputs rowsets in the same order it gets queries, regardless of what time they take.

`ordered` will not work unless `autoOrdering` is disabled.

## AutoOrdering <a id="auto-ordering"></a>
The `autoOrdering` component option groups the output sending. (_default is `true`_)

`autoOrdering` temporarily enables `ordered` for components still having `ordered: false` to make them stream-safe.

If order is important, it can be disabled by setting `component.autoOrdering = false`. [See an example of autoOrdering](https://github.com/aretecode/noflo-packets/blob/62c5161a21612e8c3d2f4f08eadd1a3b3cc0af1a/components/FilterByValue.coffee)

What `autoOrdering` does is automatically turns `ordered` on when it sees a stream coming, so it makes sure (or at least tries to) that the output stream is the result of processing exactly the same sequence as the input stream.




----------------------------------------
# Buffer <a id="buffer"></a>

If you need to do something advanced and the [Get](#Get) and [Stream](#Stream) helpers cannot do what you need, you can read information right from the buffer. To do that easily, there are `input.buffer` helpers.

<div class="note">
When you manually read from the buffer, it is not reset automatically, so you have to manually change the buffer when you are <a href="done">finished processing and are done</a>.
</div>

To get the current buffer:

```coffeescript
currentBuffer = input.buffer.get()
```

To get the current buffer for a specific port:

```coffeescript
currentInBuffer = input.buffer.get 'in'
```

To get the current buffer for a multiple ports:

```coffeescript
[inBuffer, ehBuffer] = input.buffer.get 'in', 'eh'
```

To find IPs matching criteria for a certain port:

```coffeescript
openBracketAndDataForIn = input.buffer.find 'in', (ip) ->
  (ip.type is 'data' or ip.type is 'openbracket') and ip.data?
```

To completely reset the buffer:

```coffeescript
input.buffer.set portname, []
```

or based on your conditions (in this case, keeping only ips with data type):

```coffeescript
input.buffer.filter portname, (ip) -> ip.type is 'data'
```

An example usage would be to not reset one port buffer while you reset different one and trigger on every IP.

```coffeescript
noflo = require 'noflo'

exports.getComponent = ->
  c = new noflo.Component
    icon: 'gear'
    inPorts:
      eh:
        datatype: 'all'
        required: true
      igloo:
        datatype: 'all'
    outPorts:
      canada:
        datatype: 'object'
    process: (input, output) ->
      return unless input.hasStream 'eh'
      stream = input.getStream 'eh'
      streamData = stream.filter (ip) -> ip.type is 'data'

      data = 'ehs=' + streamData.length
      if input.has 'igloo'
        igloos = input.buffer.find 'igloo', (ip) -> ip.type is 'data'
        for igloo in igloos
          data += '&' + igloo.data

      console.log data
      output.sendDone canada: data

c = exports.getComponent()
eh = new noflo.internalSocket.createSocket()
igloo = new noflo.internalSocket.createSocket()
c.inPorts.eh.attach eh
c.inPorts.igloo.attach igloo

eh.send new noflo.IP 'openBracket'
eh.send 'eh?'
eh.send 'eh!'
eh.send 'eh!?'
igloo.send 'cold'
igloo.send 'message'
eh.send 'eh'
eh.send new noflo.IP 'closeBracket'

# @TODO: NEEDS A FIX
eh.send new noflo.IP 'closeBracket'

eh.send new noflo.IP 'openBracket'
eh.send '...eh...'
eh.send new noflo.IP 'closeBracket'

# @TODO: NEEDS A FIX
eh.send new noflo.IP 'closeBracket'
```
