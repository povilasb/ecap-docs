==============
eCap Interface
==============

::

    eCAP is a software interface that allows a network application, such as an
    HTTP proxy or an ICAP server, to outsource content analysis and adaptation to a
    loadable module.

[#f3]_

Service
=======

`Service` represents eCAP adapter/plugin.

`Adapter::Service` objects should persist throughout the host application
lifetime.

The host application calls adapter::Service::start() after calling
Adapter::Service::configure() and before making the first
Adapter::Service::makeXaction() call.
The host application also calls Adapter::Service::start() if it wants to
resume using a previously stopped Service (i.e., after the
Adapter::Service::stop() call).

A typical adapter::Service call sequence (without taking reconfiguration,
suspension, and retirement into account) looks like this::

    Service()
    configure()
    start()
    wantsUri() && makeXaction()
    wantsUri() && makeXaction()
    wantsUri() && makeXaction()
    ...
    stop()
    ~Service()

[#f1]_

Xaction
=======

You MAY accumulate vb chunks in adapter::Xaction::noteVbContentAvailable().
You SHOULD NOT try to analyze the virgin body in adapter::Xaction::start()
because the body MAY NOT be available at that time (it may still be coming from
the client or server).

[#f2]_

Body Modifying Adapters
=======================

Flow::

    Request arrives to host application
    ...
    Service.start()
    Service.abMake()
    Service.noteVbContentAvailable()
    Service.abContent()
    Service.abContentShift()
    Service.noteVbContentDone()
    Service.abContent()
    Service.stop()

Asynchronous Adapters
=====================

In a nutshel, host application event loop repeatedly calls back `Service`
instance::

    +------+
    | Host |   calls every      +------------------+
    | loop |------------------> | Service.resume() |
    +------+  N time interval   +------------------+

`libecap::Adapter::Service` class has several methods specifically for
asynchronous adapters:

* `makeAsyncActions()`
* `resume()`

If you take a look at async adapter sample, you'll see that `Service`
holds a list of `Xaction` objects that are ready to be processed.
`Session.resume()` checks that list and resumes appropriate `Xaction`
instances.

More detailed info about async adapters can be found in :doc: `here <async>`.

Glossary
========

.. glossary::

    host application
        Software that integrates eCap library thus allowing to extend how HTTP
        messages are handled.

    request satisfaction
        Producing responses instead of adapting virgin requests. Meaning
        respond to request immediately without actually relaying request to
        target server.

    vb
        Virgin body - HTTP body before modifications. You will often find this
        acronym in code.

    ab
        Adapted body - HTTP body after modifications. You will often find this
        acronym in code.

.. rubric:: References

.. [#f1] https://answers.launchpad.net/ecap/+question/277300
.. [#f2] https://answers.launchpad.net/ecap/+question/136356
.. [#f3] http://e-cap.org/
