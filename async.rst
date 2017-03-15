==============================
Asynchronous eCAP Adapters API
==============================

This document describes APIs that allow eCAP adapters to run in parallel with
the host application. This is useful when an adapter has to do I/O, perform
long content analysis, or use 3rd party libraries that do those things.


For the impatient
=================

You should not rush when working with threads, but if you want to learn the
hard way, please review how adapter_async from the official eCAP sample adapter
collection implements the following methods:

  bool Adapter::Service::makesAsyncXactions() const;
  void Adapter::Service::suspend(timeval &timeout);
  void Adapter::Service::resume();
  void Adapter::Xaction::resume();

Please remember that all libecap methods have to be executed inside the host
application thread (or equivalent). Adapter code may respond to application
requests but may not call the application proactively, from the adapter
thread.

Terminology
===========

This document uses the terms "threaded" and "asynchronous" interchangeably,
even though the latter one is more general and the discussed APIs are meant to
work in any environment that supports parallel execution of multiple code
paths.

Overall design
==============

Starting with v1.0, eCAP has APIs to isolate the host application (which may
be threaded or monolithic) from a threaded adapter. The overall design is based
on the idea that one side (the host application) has to be in charge for the
isolation to work. This means that the host should poll the adaptation service
for transactions that became ready. Polling involves three steps:

Wait: The host application has to give the adapter control before the host
    starts sleeping or waiting for I/O so that the adapter can influence the
    length of that wait. This is done by calling Service::suspend().
    Host applications that do not block for a long time should still call this
    method to learn how soon they should ask the adapter about transactions
    that became ready.

Ask: After the wait described above is over, the host application gives the
    adapter control again so that the adapter can tell the host which
    transactions became ready while the host was waiting. This is done by
    calling Service::resume().

Answer: When the host asks about ready transactions (see above), the
    adaptation service can call host::Xaction::resume() and the host will,
    sooner or later, resume working on that transaction by calling its
    adapter::Xaction::resume() method. Why should not the adapter call its own
    transaction's resume() method instead? The host may need to prepare
    internal transaction state for calls such as host::Xaction::useVirgin()
    and doing so for all pending transactions may be prohibitively expensive.

This approach introduces a delay between the time an asynchronous adapter
transaction becomes ready and the time the host application resumes that
adapter transaction. On an idle or threaded server, the adapter can limit that
delay. On a busy monolithic server, the delay should be negligible (because
such a server virtually never sleeps and can poll the adapter after every
"cycle" of work).

Service::suspend() and Service::resume() calls order and timing
===============================================================

It is important to note that the Service::suspend() method may be
called independently of the Service::resume() method. The method calls
may not be "paired" in any way, even though it is convenient to think of the
methods that way. Depending on the host application implementation and needs,
either method may be called multiple times before or after the other.

In other words, the adapter must be prepared to receive a Service::suspend()
or Service::resume() call at any time, regardless of the call history.

void Service::suspend(timeval &timeout)
=======================================

If the adapter wants the next polling to happen after the specified timeout,
it should not modify the parameter value. Otherwise, the adapter should
decrease the timeout value according to the service preferences. These rules
are meant to make the host application code simpler and faster because the
host may have to iterate several services and because such iterations may run
inside a performance-critical code section. It is a good idea to optimize this
method implementation.

The initial timeout value is determined by the host application and may be as
large as the maximum value allowed by the timeval structure. Note that the
timeout value never goes up!

For example, if a Service decreases timeout to 300ms, then the adapter is
likely to call Service::resume() in about 300ms from the current time.

Again, there is no guarantee that the Service::resume() method will be
called around the timeout expiration time. For example, other adaptation
services may decrease the timeout, prompting the host application to call
Service::resume() a lot sooner, or the host may be overloaded and call
it later.

void Service::resume()
======================

This method is meant for resuming adaptation transactions that were waiting
for the host application to become active. However, the method must not call
their adapter::Xaction::resume() method directly. Instead, it should call the
corresponding host::Xaction::resume() method and let the host code prepare for
and perform transaction resumption at the host application convenience.
Moreover, a Service::resume() implementation should not call other
host::Xaction methods as the host application may not be ready to handle them.

Needless to say, the adapter has to remember transactions that are ready to be
resumed. The set of remembered transactions may have to be adjusted if the
host application stops any of them by calling adapter::Xaction::stop(). The
adapter must not attempt to resume stopped transactions.

This method may resume zero or more transactions.


bool Service::makesAsyncXactions() const
========================================

The adapter service must return true if and only if it wants to participate in
asynchronous polling. The host application calls this method at least once
after Service::start() and before sending any transactions to the service.

This method is an optimization -- without it, the host application would have
to poll all services while many adapters do not require the associated
overheads.


Storing asynchronous transactions
=================================

Asynchronous adapters must store [pointers to] pending transactions because
their state may change before the host application calls any of the
transaction methods. If keeping pending transaction state around is required,
use libecap::shared_ptr and the transaction object will not be destroyed until
both the host application and the adapter no longer use it. Otherwise,
consider using libecap::weak_ptr to get protection from dangling pointers
without delaying transaction object destruction by the host application.

If you decide to store raw transaction pointers, your code must make sure
those raw pointers are properly invalidated if the host application calls
adapter::Xaction::stop() and destroys the transaction object. Doing so
correctly is especially tricky inside asynchronous code, so be careful.

The above does _not_ imply that libecap smart pointers magically solve
thread-safety problems. The thread-safety of TR1 equivalents of
libecap::shared_ptr and libecap::weak_ptr classes is discussed in [1] and [2].
In short, they are as thread-[un]safe as built-in C++ types.

[1] http://gcc.gnu.org/onlinedocs/libstdc++/manual/shared_ptr.html
[2] www.boost.org/doc/libs/release/libs/smart_ptr/shared_ptr.htm#ThreadSafety
