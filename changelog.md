o----------
# 2016-06-20

## /changelog
- created

## /index
- updated 404 links from /library to github

## /components
- moving async component information from /components to /legacy
- adding CommonJS link
- move subgraphs to /graphs

## /legacy
- comparison of WirePattern vs Process

## /process-api
- changed IA order so component states are higher up
- changing xml-esque firing pattern example to sequentially ordered IPs
- splitting processing `getting` and `sending`
- fixed stram typo

## /graphs
- created /graphs
- moved /fbp & /json to folders as indexes so they will not appear in the sidebar, now they are merged into /graphs
- add a more simple example of using subgraphs

## package.json
- added license field & repo link

## readme.md
- updating required gems

## Gruntfile.coffee
- removed imgmin
- comment out 'docco' from task: 'build' because of error `Fatal error: spawn pygmentize ENOENT`

## /publishing
- changing example link from [noflo-basecamp](https://github.com/noflo/noflo-basecamp) to [noflo-core](https://github.com/noflo/noflo-core) and [ingres-table](https://github.com/c-base/ingress-table)
- change `component.json` link from depreciated in 2014, to latest

## /glossary
- adding CommonJS link

# canadianness

## spec/
- first round at fbp-spec, long error.

----------
# 2016-06-21

## /components & /process-api
- moved port attributes from /process-api to /components/#port-attributes
- adding IP info
- reordering index

## *
- changing 'did you know' blocks from <blockquote> to jekyll styled <div>s

## /debugging
- created

## /testing
- removed for now, should it just reference writing-projects guide?

## process-api
- fix component-states header link
- use destructuring when getData with multiple
- changing xml of per packet to be flow order same as full stream

----------
# 2016-06-22

## /process-api
- changing null brackets to use default
- fix scope header
- expand scope content

## /writing-projects
- linking to noflo-core & InternalSockets

## /process-api -> /testing
- moving bracket forwarding example so it will be replaced with an animation into testing to be used as component loading example

## /components#portevents
- added IP

## config
- added redcarpet extension

## css
- added table styles

## legacy
- table md
- translation extension

## graphs
- fixing json anchor
- change blockquote to notes div
- make order of code (js, cs) consistent

## writing projects
- make writing your own projects in its own folder
- make initial layout for it

----------
# 2016-06-23

## writing projects
- adding page weights and sorting in order
- adding links to the repo for each part and links to next part of the guide
## FindEhs
- expanding on FindEhs
## graphs
- linking to subgraphs docs

## process-api
- adding note about `control` ports dealing with only data

## graphs
- changing subraph punctuation
- adding flowhub section

## package.json
- creating

## projects/embedding
- creating
- rewrite index of canadianness and document it
- add ss example of flowtrace and instructions on using it

## projects/summary
- splitting from index of /projects

## hidden
- added async-components, fbp, json, and protocol back as files, then hid them from the documentation navigation

----------
# 2016-06-24

## graphs
- screenshots

## testing
- codeblock highlighting
- adding notes about noflo requirements for using special cases

## publishing
- add publishing NPM with Travis guide
- remove old parts
- add index

## glossary
- add ip

## nav
change `components` link from https://www.npmjs.com/browse/keyword/noflo to https://https://npmjs.com/browse/depended/noflo so it shows in order of recent and used

## testing
- adding component loading inline and link to it from bracket pooling

## process-api
- testing & fixing the buffer example

# 2016-06-25

## process-api
- changed todo to processerror in #done

## projects/#word-score- updating

## legacy
- formatting

## url
- putting on github pages
- changing baseurl for github pages
- change projects link to relative

-----------
# 2016-06-26

- reading the docs

## legacy
- formatting headers in porting
- updating tested version of ported filtervalue

-----------
# 2016-06-26

## process-api/#brackets
- added how they should begin & end with the same data

-----------
# 2016-06-28

## process-api/#auto-ordering
- added in blurb about autoOrdering

-----------
# 2016-06-29

## components
- fixed Payload
- moving port events to legacy
- adding stream datatype
- adding triggering

## process-api
- fix `until` statement
- clarify `input.get`
- fixed #buffer Removing statement
- flushing out #stream-helpers
-

## writing your own projects
- adding file structure
- removed
<div class="note">
 When you send an IP, if you do not send an openBracket and closeBracket explicitly, or let the forwardBrackets do the wrapping for you, every packet you send will emit its own openBracket and closeBracket.
</div>


-----------
# 2016-06-30

## general
- changing `thing` and `stuff` to use concrete terms

## nav
- adding page weight to documentation pages
- and support for weights in documentation layout
- making /projects pages be subpages of /documentation

## /information-packets
- creating page from https://github.com/noflo/noflo/issues/290
- adding scopes examples
- adding a paragraph explaining concurrency issues within a graph and how scope solves it.

## /process-api
- fixing `get` by using until
- clarifying scope concurrency - moving to /information-packets
- adding forwardBrackets example

## /testing
- moving to /projects/testing

## /projects/index/#file-structure
- flushing it out more


-----------
# 2016-7-01

## /information-packets
- remove outdated bit

## /legacy
- remove incorrect information
<!--
### Buffer
WirePattern | Process
| --- | --- |
| if it is ordered (which it is by default), if you have for example `in: ['one', 'two']` and `two` gets data before `one`, the packet for `two` will be dropped. Then if a packet comes to `one` and the next one comes to `two`, it will be triggered. | every packet that comes in is appended to the buffer for the port it was sent to, when you use `get*` functions or manually access the buffer, some or all data will be removed from the buffer. |
-->

## /components/#port-attributes
- flush out control & triggering

## /process-api/#ordering
- add details to ##ordered & ##auto-ordering

## nav
- adding styles to make subpages indented

## /projects - components
- use short version

## summary

-----------
# 2016-07-03

## /process-api
- adding section on dataStream
- add multiple arguments for getStream, hasStream, buffer.get
- add advanced forwardBrackets example to show how it works

## /information-packets
- updating #data, #open-brackets, and #close-brackets

## projects
/#index
- add Gruntfile link
/#file-structure
- change formatting
- fix fbpspec file definition
/#determine-emotion
- lower header level for #2
/#testing
- add link to fbpspec.coffee
/#package-json
- link to canadianness package.json

## legacy
- updating index
- fix `use` list formatting

## /sitemap
- creating

## footer
- adding sitemap link

## nav
- fixing index page on documentation when active

-----------
# 2016-07-04

/components
- fixing formatting in multiple attributes

/projects
- fixing grunt link
- fixing researching grammar

/nav
- igloring sitemap in nav on /projects

/process-api
- all port stuff events -> change

-----------
# 2016-07-05

/ports
- create, move from /components
- add sockets
- add sockets animation
- add sockets sending methods

/process-api
- flush out intro to process api
- add components states
- adding when to use IPs
- added why not to use sub-streams
- flush out how data stream hasDataStream checks behind the scenes

# general
- use site.baseurl for images
