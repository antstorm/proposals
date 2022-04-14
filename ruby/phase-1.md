# Ruby SDK - Phase 1

Ruby is a popular dynamic programming language that many companies are leaning on to beat their
competitors to the market. With the focus on the speed of execution it is very common for these
companies to rely on managed solutions. This presents an opportunity for Temporal to become Ruby's
de facto standard for workflow orchestration.

This proposal aims to adhere to the Ruby way of writing code favouring convention over
configuration, readability, testability and developer's productivity. These are prioritised in order
to avoid the destiny of the [AWS Flow Framework for Ruby](https://github.com/amazon-archives/aws-flow-ruby)
that never appealed to the Ruby community.


## Disclaimer

This proposal draws substantial inspiration from the [Temporal Ruby SDK](https://github.com/coinbase/temporal-ruby)
developed at Coinbase by the author of this proposal. This allows us to pick the best parts of the
former SDK and improve on the parts that are not so good.

The proposal itself follows the [Python SDK proposal](https://github.com/temporalio/proposals/blob/master/python/phase-1.md)
for the same exact reason.


## Timeline

The Ruby SDK will be implemented in four loosely defined phases:

- Phase 1 — initial setup, execution attributes, workflow/activity outer APIs, client and worker
- Phase 2 — workflow and activity inner APIs, local activities, async activity completion, etc
- Phase 3 — payload converters, error handling, middleware
- Phase 4 — Sandbox, replayer, support for testing


## Overview

The Ruby SDK will be implemented on top of the [SDK Core](https://github.com/temporalio/sdk-core).
The exact bridging mechanism will decided later as there are different alternatives.

### Goals

- Quickest path to a working setup
- Convention over configuration
- Avoid sacrificing configurability of a complex setup
- Minimise external dependencies (gems)
- Local and end-to-end testability of workflows

### Non-goals

- Replicate the interface of other SDKs
- Backwards compatibility with Coinbase's Ruby SDK

### Tech stack && approach

- Ruby >= 2.7 (currently oldest fully supported version)
- Packaged as a Ruby gem (and published to [https://rubygems.org])
- FFI or Rutie bridge into SDK Core (to be decided)


## An end-to-end example

Perhaps a bit unconventionally (as far as the proposal review process goes), but we'd like to start
with an end-to-end example of a simple and fully functional "Hello World" workflow. The intention of
this example is to demonstrate how everything ties together before diving into each separate moving
part and exploring it in depth.

```ruby
require 'temporal'
require 'temporal/worker'

Temporal.configure do |config|
  config.url = 'localhost:7233'
  config.namespace = 'my-namespace'
  config.task_queue = 'my-task-queue'
end

class MyActivity < Temporal::Activity
  def execute
    puts 'Hello World!'
  end
end

class MyWorkflow < Temporal::Workflow
  def execute
    MyActivity.execute!
  end
end

worker = Temporal::Worker.new
worker.register_workflow(MyWorkflow)
worker.register_activity(MyActivity)
worker.run

# Executed from a separate process/thread to kick off the workflow
Temporal.start_workflow(MyWorkflow)
```

While probably a bit too simple for an average use-case, this is the first thing that most engineers
will be looking at when considering using Temporal at their companies. Hence the simplicity and a
cheeky attempt to make a good first impression.


## Configuration

In order to make all subsequent interactions with the Ruby SDK less explicit (an explicit example is
available further down in this proposal) it's beneficial to have the configuration defined upfront.

Here's an example of a global application configuration:

```ruby
Temporal.configure do |config|
  config.url = 'my-temporal-server.com:7233' # connection string
  config.namespace = 'my-namespace' # default namespace unless explicitly overridden by the caller
  config.task_queue = 'my-task-queue' # default task queue unless explicitly overridden by the caller
  config.logger = Logger.new(STDOUT, progname: 'temporal-ruby-sdk') # default Ruby logger
end
```

Notes:

- This configuration will get picked by the singleton client
- This configuration will be used by the worker (unless explicitly overridden)
- This is not the full list of available configuration options
- All the configuration options come with sane defaults

For use-cases that require more flexibility, a configuration can be explicitly created for each
client and worker:

```ruby
config_1 = Temporal::Configuration.new(
  url: 'host-1.com:7233',
  namespace: 'namespace-1',
  task_queue: 'queue-1'
)
client_1 = Temporal::Client.new(config_1)
worker_1 = Temporal::Worker.new(config_1)

config_2 = Temporal::Configuration.new(
  url: 'host-2.com:7233',
  namespace: 'namespace-2',
  task_queue: 'queue-2'
)
client_2 = Temporal::Client.new(config_2)
worker_2 = Temporal::Worker.new(config_2)
```


## Activity/Workflow execution attributes

This is an important and novel (as far as all the other SDKs are concerned) concept that allows to
reduce the explicitness of using the Ruby SDK. This is achieved via a few assumptions that have been
tested against the real world use-cases, but obviously are up for a debate. These are:

- The simplest __viable__ use-case can be satisfied with a single namespace and single task queue
- Most workflow/activity names are identical to the names of the classes implementing them
- The most popular setup will be based on a one-to-one mapping between workflows/activities and a
  namespace + task queue combination
- In most cases timeouts and retry policies are defined per workflow/activity and not a specific
  execution. That is to say that workflow/activity owners will probably have the best idea of which
  timeouts and retry policies should be used when executing it

With that in mind we're proposing a way to configure the said attribute per workflow/activity
definition. This would reside in the outer API and would look something like this:

```ruby
class NewWorkflow < Temporal::Workflow
  namespace 'ns-1'
  task_queue 'tq-1'
  timeouts run: 10 * 60 * 60, execution: 20 * 60 * 60

  def execute
    ...
  end
end

class AnotherActivity < Temporal::Activity
  namespace 'ns-2'
  task_queue 'tq-2'
  retry_policy(
    interval: 1,
    backoff: 1,
    max_attempts: 3,
    non_retriable_errors: [FatalError]
  )

  def execute
    ...
  end
end
```

The main idea here is when these are started/scheduled the caller doesn't have to explicitly specify
`name`, `namespace`, `task_queue`, `timeouts` or `retry_policy`, but can instead rely on the
existing definitions. The same exact concept applies to registering these classes with a worker,
that can then figure out which namespace and task queue need to be polled. Here's what that looks
like:

```ruby
# starting a workflow
client.start_workflow(NewWorkflow) # assumes namespace: 'ns-1', task_queue: 'tq-1'

# registering classes with a worker
worker.register_workflow(NewWorkflow) # assumes namespace: 'ns-1', task_queue: 'tq-1'
worker.register_activity(NewActivity) # assumes namespace: 'ns-2', task_queue: 'tq-2'
```

This of course can be fully or partially overwritten by the caller, which has the full control over
the resulting attributes. For example:

```ruby
client.start_workflow(NewWorkflow, options: { namespace: 'ns-3' }) # namespace: 'ns-3', task_queue: 'tq-1'

worker.register_workflow(NewWorkflow, task_queue: 'tq-3') # namespace: 'ns-1', task_queue: 'tq-3'
worker.register_activity(NewActivity, namespace: 'ns-3', task_queue: 'tq-3') # namespace: 'ns-3', task_queue: 'tq-3'
```

Pushing this even further offers the same flexibility as any other Temporal SDK:

```ruby
client.start_workflow(
  'my-custom-workflow',
  options: {
    namespace: 'ns-4',
    task_queue: 'tq-4'
  }
) # all params are explicit

worker.register_workflow(
  NewWorkflow, # used only as an implementing class
  name: 'my-custom-workflow',
  namespace: 'ns-4',
  task_queue: 'tq-4'
) # no assumptions
```

The resulting attributes are decided following this hierarchy (most important are higher up):

1. Attributes that are passed explicitly by the caller
2. Attributes defined in workflow/activity classes
3. Attributes specified in the configuration for the client/worker
4. Default attributes (only when applicable)

Notes:

- This approach does not enforce class definitions for external workflows/activities (e.g.
  implemented using a different SDK)
- Timeouts and retry policy are omited from the caller examples, but work exactly the same
  way
- When scheduling activities we'll use the same approach. More on this in the Phase 2, when
  we'll be discussing workflow and activity inner APIs
- Hopefully this will explain how the above-mentioned Hello World example works


## An explicit end-to-end example

Here we'd like to show that while the Ruby SDK users can lean on the default assumptions, they still
have all the explicit options available to them. This example is functionally identical to the Hello
World example from above.

```ruby
require 'temporal'
require 'temporal/worker'

NAMESPACE = 'my-namespace'
TASK_QUEUE = 'my-task-queue'

class MyActivity < Temporal::Activity
  def execute
    puts 'Hello World!'
  end
end

class MyWorkflow < Temporal::Workflow
  def execute
    workflow.execute_activity!(
      'MyActivity',
      options: {
        namespace: NAMESPACE,
        task_queue: TASK_QUEUE
      }
    )
  end
end

config = Temporal::Configuration.new(url: 'localhost:7233')

worker = Temporal::Worker.new(config)
worker.register_workflow(
  MyWorkflow,
  name: 'MyWorkflow',
  namespace: NAMESPACE,
  task_queue: TASK_QUEUE
)
worker.register_activity(
  MyActivity,
  name: 'MyActivity',
  namespace: NAMESPACE,
  task_queue: TASK_QUEUE
)
worker.run

# Executed from a separate process/thread to kick off the workflow
client = Temporal::Client.new(config)
Temporal.start_workflow('MyWorkflow', options: { namespace: NAMESPACE, task_queue: TASK_QUEUE })
```

*NOTE: While this example shows the extreme end of explicitness, it is a spectrum and the SDK users
can choose to gradually start overriding the defaults until they achieve what feels comfortable.*


## Client

The client is the primary point for interacting with workflows and async activities as well as
managing namespaces. As hinted in the previous section there's a global singleton client available
for applications that only connect to a single Temporal server and don't need varying
configurations:

```ruby
# Global instance of the Client configured via `Temporal.configure`
Temporal.start_workflow(MyWorkflow)

# Explicitly configured local instance of the Client
config = Temporal::Configuration.new(url: 'localhost:7233')
client = Temporal::Client.new(config)
client.start_workflow(MyWorkflow)
```

Despite the differences both clients offer exactly the same interface (the singleton client is an
instance of a regular client stored on Temporal module for convenience). The interface will look
something like this:

```ruby
class Client
  def initialize(config)

  def start_workflow(
    workflow, # workflow class or a name
    *args, # any args to be passed as the input to the workflow
    options: {}, # execution options overrides
    **kwargs # keyword arguments as inputs are also supported
  ) -> WorkflowHandle

  def workflow_handle(namespace, workflow_id, run_id = nil) -> WorkflowHandle
end
```

Notes:

- This is not a complete definition of the client
- Both named and keyword arguments are supported, except for the reserved `options:`

The combination of the singleton client and implicit execution attributes allows us to optionally
support direct workflow invocations (removing all SDK traces out of the way):

```ruby
MyWorkflow.start(arg_1, arg_2, keyword_arg: arg_3)
```

Once a workflow is started all the interactions will be going through the handle object returned to
the caller. The handle can of course be created from a namespace/workflow_id/run_id combination
using the `Client#workflow_handle` method. Here the interface of the handle:

```ruby
class WorkflowHandle
  def initialize(connection, namespace, workflow_id, run_id = nil)

  def result(await: false, timeout: nil) -> Any

  def describe -> WorkflowExecutionInfo

  def cancel -> void

  def query(name, *args) -> Any

  def signal(name, *args) -> void

  def terminate(reason, details: nil) -> void
end

```


## Worker

The worker interface allows SDK users to register any number of both workflows and activities. These
will then be polled for on the specified namespaces & task queues and executed. The worker can poll
on multiple task queues at the same time. These will be determined by the registered activities and
workflow. If there are any overlaps we'll only create one poller for each unique task queue.

```ruby
config Temporal::Configuration.new(url: 'localhost:7233')
worker = Temporal::Worker.new(config)

worker.register_workflow(NewWorkflow)
worker.register_workflow(NewWorkflow, task_queue: 'tq-3')
worker.register_activity(AnotherActivity)
worker.register_activity(AnotherActivity, namespace: 'ns-3')

# Calling Worker#run will block the main thread until receiving SIGTERM or SIGINT
worker.run
```

Notes:

- The worker will poll for workflows on `ns-1`/`tq-1` and `ns-1`/`tq-3`
- And activities on `ns-2`/`tq-2` and `ns-3`/`tq-2`
- The workflows/activities can of course be defined in any number of different workers
- The worker is intended to take over the process

### Concurrency model

Ruby can be a tricky language for running concurrent applications. It supports kernel threads,
however it has a Global Interpreter Lock (GIL) in place which prevents any two threads to be
executed at the same time (making it effectively single-threaded). The good news is that GIL does
allow IO-bound instructions to be executed in parallel.

Because of these limitations the most resource efficient solution is a combination of process
forking and threading. The exact ratio can depend heavily on the application and hardware used. This
is why it is crucial to allow the Ruby SDK users to configure how many processes/threads they need
per each worker.

Internally a worker will run each poller in its own thread and will create 2 shared thread pools of
a given size for executing tasks (one pool for activities and one for workflows). A poller will
reserve a thread on a thread pool before polling to ensure there's capacity for processing a task.

To support multiple processes we'll have a `Temporal::Crew` class that run a given worker
across a defined number of forked processes. The parent process will then monitor child processes
restarting misbehaving and dead ones.

```ruby
worker = Temporal::Worker.new(config)
crew = Temporal::Crew.new(worker, 16)

crew.dispatch
```
