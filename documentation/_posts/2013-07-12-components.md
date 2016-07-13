---
layout: documentation
title: Components
categories:
 - documentation
weight: 3
---

- [Structure](#structure)
- [Lifecycle](#lifecycle)
- [SubGraphs](#subgraphs)
- [Design](#design)

[picture](ingredient)

A component is the main ingredient of flow-based programming. Component is a [CommonJS module](http://requirejs.org/docs/commonjs.html) providing a set of input and output port handlers. These ports are used for connecting components to each other.

[picture](box)

NoFlo processes (the nodes of a flow graph) are instances of a component, with the graph controlling connections between ports of components.

Since version 0.2.0, NoFlo has been able to utilize components shared via NPM packages. [Read the introductory blog post](http://bergie.iki.fi/blog/distributing-noflo-components/) to learn more.

<a id="structure"></a>
### Structure of a component

Functionality a component provides:

* List of inports (named inbound ports)
* List of outports (named outbound ports)
* Handler for component initialization that accepts configuration (the `exports.getComponent = ->` that a `componentLoader` or graph will pass metadata to)
* Handler for connections for each inport (Proccess API)

Minimal component written in NoFlo would look like the following:

```coffeescript
# File: components/Forwarder.coffee
noflo = require 'noflo'

exports.getComponent = ->
  component = new noflo.Component
    description: 'This component receives data on a single input
    port and sends the same data out to the output port'

    # Register ports
    inPorts:
      in:
        datatype: 'all'
    outPorts:
      out:
        datatype: 'all'

    # Our event handler
    process: (input, output) ->
      if input.has 'in'
        # Forward data when we receive it.
        output.sendDone out: input.getData 'in'
```


[animation]()

This example component register two ports: _in_ and _out_. When it receives data in the _in_ port, it sends the same data to the _out_ port. So basically this component would be a simple repeater.

The `exports.getComponent` function is used by NoFlo to create an instance of the component. See [Component Lifecycle](#lifecycle) for more information.

You can find more examples of components in the [component library](/library/) section of this website.

<a id="subgraphs"></a>
### moved to [Subgraphs](/documentation/graphs/#subgraphs)

<a id="design"></a>
### Some words on component design

Components should aim to be reusable, to do one thing and do it well. This is why often it is a good idea to split functionality traditionally done in one function to multiple components. For example, counting lines in a text file could happen in the following way:

<!-- It would be interesting to have an image here with the graph described
below -->

* Filename is sent to a _Read File_ component
* _Read File_ reads it and sends the contents onwards to _Split String_ component
* _Split String_ splits the contents by newlines, and sends each line separately to a _Count_ component
* _Count_ counts the number of packets it received, and sends the total to a _Output_ component
* _Output_ displays the number

This way the whole logic of the application is in the graph, in how the components are wired together. And each of the components is easily reusable for other purposes.

If a component requires configuration, the good approach is to set sensible defaults in the component, and to allow them to be overridden via an input port. This method of configuration allows the settings to be kept in the graph itself, or for example to be read from a file or database, depending on the needs of the application.

The components should not depend on a particular global state, either, but instead attempt to keep the input and output ports their sole interface to the external world. There may be some exceptions, like a component that listens for HTTP requests or Redis pub-sub messages, but even in these cases the server, or subscription should be set up by the component itself.

When discussing how to solve the unnecessary complexity of software, _Out of the Tar Pit_ promotes an approach quite similar to the one discussed here:

> The first thing that we’re doing is to advocate separating out all complexity of any kind from the pure logic of the system (which - having nothing to do with either state or control - we’re not really considering part of the complexity).

Done this way, components represent the pure logic, and the control flow and state of the application is managed separately of them in the graph. This separation makes the system a lot simpler.

<a id="lifecycle"></a>
### Component lifecycle

When NoFlo is being run, all components used in a NoFlo network (an instantiated graph) go through the following lifecycle steps:

* _Instantiation_: component is instantiated for a node in the NoFlo graph, and its `constructor` method is called. At this stage a component should prepare its internal data structures and register listeners for the [events of its input ports](#portevents)
* _Running_: component has received events on its input ports. Now the component can interact with those events and the external world, and optionally start transmitting [port events](#portevents) on its output ports
* _Shutdown_: normally a NoFlo network finishes when all components stop transmitting port events. It is however also possible to close a running NoFlo network. In this case the `shutdown` method of components gets called, allowing components to perform whatever cleanup needed, like unregistering event listeners on external objects, or closing network connections.

A running instance of a component in a NoFlo network is called a *process*. Before a process has received data it should be *inert*, merely listening to its input ports. Processes that need to start doing something when a network is started should be triggered to do so by sending them an Initial Information Packet (IIP).

<a id="icons"></a>
### Component icons

For use in visual editors like [Flowhub](http://flowhub.io/), components can provide an icon. The icons are based on the [Font Awesome icon set](http://fontawesome.io/icons/), but without the `fa-` prefix.

A component that takes a picture could for instance use the [Camera icon](http://fontawesome.io/icon/camera/). The icons are declared using the `icon` property of the component:

```coffeescript
# File: components/TakePicture.coffee
exports.getComponent = ->
  component = new noflo.Component
  component.description = 'Take a photo with the computer\'s web camera'
  component.icon = 'camera'
```

Icons can also be updated during runtime to reflect a changing state of the component. This is accomplished by calling the `setIcon` method of the component. For example, the *TakePicture* component above could temporarily set its icon when a picture has been taken to a [Picture icon](http://fontawesome.io/icon/picture-o/) and then change it back a bit later:

```coffeescript
component.originalIcon = component.getIcon()
component.setIcon 'picture-o'
component.timeout = setTimeout =>
  component.setIcon component.originalIcon
  component.timeout = null
, 200
```

