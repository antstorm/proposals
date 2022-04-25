# Ruby SDK - Phase 1

Ruby is a popular dynamic programming language that many companies are leaning on to beat their
competitors to the market. With the focus on the speed of execution it is very common for these
companies to rely on managed solutions. This presents an opportunity for Temporal to become Ruby's
de facto standard for workflow orchestration.


## Timeline

The Ruby SDK will be implemented in four loosely defined phases:

- Phase 1 — initial setup, activity APIs, client and worker, payload converters
- Phase 2 — per-class execution attributes, logging, metrics, error handling, middleware
- Phase 3 — workflow APIs, local activities, async activity completion, etc
- Phase 4 — Sandbox, replayer, support for testing


## Overview

The Ruby SDK will be implemented on top of the [SDK Core](https://github.com/temporalio/sdk-core).
The exact bridging mechanism will decided later as there are different alternatives.

### Goals

- Quick path to a full working setup
- A path of minimal divergence from other SDKs (does allow for other approaches, but it's not the
  goal)
- Minimise external dependencies (gems)
- Local and end-to-end testability of workflows
- Type safety where possible (optional and exact approach to be chosen later)
- Compatibility with existing tools (error handling, logging, metrics, middleware, etc)

### Non-goals

- Convention over configuration (in favour of cross-SDK interface consistency, this goes against
  Ruby principles)
- Backwards compatibility with Coinbase's Ruby SDK (current SDK customers will have to do a full
  migration)
- Reducing the number of different approaches to achieving the same goal (let the SDK user choose
  what's best for their application/taste)

### Tech stack && approach

- Ruby version >= 2.7 (currently oldest fully supported version)
- Type safety based on RBS or Sorbet (RBS will require Ruby version => 3.0)
- Packaged as a Ruby gem (and published to [https://rubygems.org])
- FFI or Rutie bridge into SDK Core (to be decided)


## Key language considerations and restrictions

We're assuming that the audience for this proposal might not be intimately familiar with Ruby. So
before we jump into the main part here are some important considerations to keep in mind to avoid
confusion during the discussion.

*NOTE: The provided examples are not implying these are correct or incorrect ways of using Ruby, but
simply demonstrate language specifics.*

### Method invocations

All parenthesis in Ruby are optional (despite the presense or absense of method arguments). So these
calls are identical:

```ruby
my_method()
my_method
```

As well as these:

```ruby
my_method(a, b, c)
my_method a, b, c
```

The most common style suggests not using parenthesis when there are no arguments and using them
otherwise. This rule is often ignored in a case of a DSL
(e.g. [RSpec](https://relishapp.com/rspec/rspec-expectations/docs/built-in-matchers)).

### Global scope

Unlike other dynamic languages Ruby does not support requiring (or importing) files into a local
scope. In practice that mean that everything that gets `require`d becomes globally available from
every single part of the application. This applies to eveything including constants, variables,
methods, classes and modules. Here's an example:

my_method.rb:

```ruby
def my_method(name)
  puts "Hello World, #{name}!"
end
```

my_class.rb:

```ruby
class MyClass
  def run
    my_method('Alice')
  end
end
```

main.rb:

```ruby
require './my_method.rb'
require './my_class.rb'

MyClass.new.run
```

Running `ruby main.rb` will print `Hello World, Alice!`.

### "Monkey" patching

In case of a variable or a method name collision the last definition overrides the previous one
without hesitation. Same principle applies to methods defined on a class, but not the class itself —
class definitions are merged together resulting in an altered class definition in the global scope.

```ruby
class MyClass
  def my_method
    puts 'Hello World!'
  end
end

class MyClass
  def my_method
    puts 'Hello Monkeys!'
  end

  def new_method
    puts 'Hello New World!'
  end
end

my = MyClass.new
my.my_method
# => Hello Monkeys!
my.new_method
# => Hello New World
```

### Concurrency

Ruby can be a tricky language for running concurrent applications. It supports kernel threads,
however it has a Global Interpreter Lock (GIL) in place which prevents any two threads to be
executed at the same time (making it effectively single-threaded). The good news is that GIL does
allow IO-bound instructions to be executed in parallel.

While threads can offer a great performance boost in applications with a significant IO reliance. In
order to effectively utilize a multi-core processor a Ruby application has to be executed using
multiple processes.

#### Fibers

Ruby also has Fibers that allow light weight cooperative concurrency. These are code blocks that can
be paused (from inside) and resumed (from outside). These are really useful for controlling the
execution of workflow code. However unlike threads Fibers are never preempted and their scheduling
has to be done manually by the code:

```ruby
fiber = Fiber.new do
  i = 0
  loop do
    i += 1
    puts i
    Fiber.yield
  end
end

fiber.resume
# => 1
fiber.resume
# => 2
fiber.resume
# => 3
```

*NOTE: Each `fiber.resume` call blocks the main thread until a Fiber `yield`s back control.*

#### Reactors

Somewhat less popular, but still very viable are libraries that implement a reactor pattern for
building concurrent applications (e.g. Async or EventMachine). These libraries allow users to
concurrently execute IO operations.

There are however quite a few big limitations to this approach:

- You need to use very specific libraries to perform IO calls this way (there isn't one for gRPC)
- Any CPU-bound instructions block the reactor and therefore need to be executed in small bursts per
  reactor tick
- Any user-defined code can block the whole thing and has to be wrapped in threads anyways


## A basic end-to-end example

Perhaps a bit unconventionally (as far as the proposal review process goes), but we'd like to start
with an end-to-end example of a simple and fully functional "Hello World" workflow. The intention of
this example is to demonstrate how everything ties together before diving into each separate moving
part and exploring it in depth.

```ruby
require 'temporal'
require 'temporal/worker'

class MyActivity < Temporal::Activity
  def execute(name)
    puts "Hello World, #{name}!"
  end
end

class MyWorkflow < Temporal::Workflow
  def execute(name)
    # This part is outside of the scope of the 1st phase of the proposal
    MyActivity.execute!(name, options: { task_queue: 'my-queue' })
  end
end

config = Temporal::Configuration.new(
  url: 'localhost:7233'
)

client = Temporal::Client.new(config)
client.start_workflow(MyWorkflow, 'Alice', options: {
  namespace: 'my-namespace',
  task_queue: 'my-queue'
})

worker = Temporal::Worker.new(config, 'my-namespace', 'my-queue')
worker.register_workflow(MyWorkflow)
worker.register_activity(MyActivity)
worker.run
```


## Client

The client is the primary point for interacting with workflows and async activities as well as
managing namespaces. A client is created with a configuration object that specifies options shared
between a worker and a client.

```ruby
config = Temporal::Configuration.new(url: 'localhost:7233')
client = Temporal::Client.new(config)
client.start_workflow(MyWorkflow, 'arg_1', kwarg_2: 'foo', options: {
  namespace: 'my-namespace',
  task_queue: 'my-queue'
})
```

Notes:

- Client is intended to be multi-tenant capable (no limit on a single namespace)
- Configuration will provide defaults where possible (avoiding extensive upfront configuration)
- Configuration will be used for other more advanced options such as logger, metrics, payload
  converters, middleware, etc

```ruby
class Temporal::Client
  # Creates a new instance of a Client
  def self.new(config: Temporal::Configuration, **options) -> Temporal::Client

  # Start the workflow and return handle for further interaction
  # TODO: expand on :options
  def start_workflow(
    workflow, # workflow class or a name
    *args, # any args to be passed as the input to the workflow
    options: {}, # execution options
    **kwargs # keyword arguments as inputs are also supported
  ) -> Temporal::WorkflowHandle

  # Start the workflow and block awaiting it's completion
  def execute_workflow(
    workflow, # workflow class or a name
    *args, # any args to be passed as the input to the workflow
    options: {}, # execution options
    **kwargs # keyword arguments as inputs are also supported
  ) -> Any

  # Generate a handle object from a namespace, workflow_id and a run_id
  def workflow_handle(namespace, workflow_id, run_id = nil) -> Temporal::WorkflowHandle
end
```

Notes:

- This is not a complete definition of the client
- Both positional arguments and keyword arguments are supported, except for the reserved `options:`
- Conventions similar to Python SDK are used here:
  - `Client` is a gneral purpose interface, not limited workflow-only interactions
  - Due to this method names are more descriptive than just `#start`, `#execute`, etc
  - `Client#execute_workflow` is added for convenience

Once a workflow is started all the interactions will be going through the handle object returned to
the caller. The handle can of course be created from a workflow_id/run_id combination using the
`Client#workflow_handle` method. Here the interface of the handle:

```ruby
class Temporal::WorkflowHandle
  # Return the result of the workflow execution or raise if workflow is still running
  # - await: true will wait (for up to :timeout seconds) for the workflow result
  def result(await: false, timeout: nil) -> Any

  # Returns more information about a workflow
  def describe() -> Temporal::WorkflowExecutionInfo

  # Sends a cancellation request
  def cancel() -> void

  # Sends a query and awaits for the response
  def query(name, *args) -> Any

  # Sends a signal
  def signal(name, *args) -> void

  # Terminates a workflow
  def terminate(reason, details: nil) -> void
end
```

### Alternatives considered

We thought about configuring the `Temporal::Client` with a namespace, similar to the Python SDK.
It does bring the benefit of avoiding namespace to subsequent calls to the client. However we feel
like it is an unjustified and artificial limitation that will impact some multi-tenant applications.
Besides this can always be brought in a shape of a `NamespacedClient` if there's a need for it.


## Worker

The worker interface allows SDK users to start processing registered workflow and activities. These
will then be polled for on the specified namespaces & task queues and executed.

```ruby
config = Temporal::Configuration.new(url: 'localhost:7233')

worker_1 = Temporal::Worker.new(config, 'my-namespace', 'my-task-queue')
worker_1.register_workflow(MyWorkflow) # uses 'MyWorkflow' as a name
worker_1.register_activity(MyActivity) # uses 'MyActivity' as a name
worker_1.start

worker_2 = Temporal::Worker.new(config, 'my-namespace', 'another-task-queue')
worker_2.register_workflow(MyWorkflow, name: 'my-workflow') # uses 'my-workflow' as a name
worker_2.register_activity(MyActivity, name: 'my-activity') # uses 'my-activity' as a name
worker_2.start

# later
worker_1.stop
worker_2.stop
```

Notes:

- This is not complete example of all configuration options for a worker
- A worker will use the workflow/activity class name as a name unless it is explicitly provided (
  `worker_2` demonstrated this approach)
- Each worker will run in its own thread
- A blocking `Worker#run` method will be available for convenience of running a single worker in
  a process until SIGTERM or SIGINT is received
- In case connection sharing is a concern we can create and memoize a connection on the instance of
  Temporal::Configuration

And here is the `Worker` interface:

```ruby
class Worker
  # Create a new instance of a Worker
  def self.new(config: Temporal::Configuration, namespace: String, task_queue: String, **options) -> Worker

  # Register a workflow with an optional :name
  def register_workflow(Class, name: nil) -> void

  # Register an activity with an optional :name
  def register_activity(Class, name: nil) -> void

  # Start the worker and block until the process exits
  def run() -> void

  # Start the worker asynchronously
  def start() -> void

  # Gracefully stop the worker (wait for all running workflow/activities to finish)
  def stop() -> void
end
```

Internally each worker will create a thread for task polling as well as a thread pool of a given
size (passed to the initializer as an optional `:threads` argument). Then each activity task will be
dispatched to an available thread (via an in-memory queue). To make sure a polling worker always has
processing capacity a polling request will only be dispatched if there is an available thread in the
pool (in other words the pool will have no buffer).

A very similar model will be used for workflow tasks, with an added complexity of wrapping each
workflow in its own Fiber (inside a thread). This serves two purposes:

1. A workflow code can be paused and resumed during the history replay
2. Sticky workflows can be left paused while waiting for the next workflow task

Given that Fibers are very lightweight this will allow us to keep many stick workflows efficiently
in memory and manage accordingly to the worker configuration.

Here's a diagram to demostrate the whole setup:

<img src="./ruby_threads.png" width="400" align="center" alt="Temporal" />

### Alternatives considered

We've decided against using the `Client` to initialize the `Worker` (something that Go and Python
SDKs are doing). It seemed like a wrong abstraction and would require making many attributes public
on the client to make sure the worker can access them (expanding the public API of a client). Using
a configuration object solves this and opens up further optimisation opportunities (that will be
proposed in the 2nd phase).

We can potentially use reactor pattern via Async gem. This has the benefit of giving extra
performance, however it would drastically reduce the performance unless activities are written in a
reactor-friendly way. This can be further added as an optional approach to apps already using Async,
but doesn't feel like a great default option.

Additionally the SDK users will want to leverage multiple CPU by spawning multiple worker processes.
We have agreed that this part should be left out of the Ruby SDK and handled by developers
themselves. Potentially this functionality can be added as a separate Ruby gem.


## Activities

Each activity is going to be represented by its own class. This decision comes with a few benefits:

- There's no need to pollute global scope with activity methods (see "Global Scope" section)
- There's no need to inject global context for communicating with Temporal. It will be available
  through a private variable on the class instance (`#activity`)
- It plays nicely with Ruby's OOP model
- It allows for future extensibility including 3rd party extensions
- Activity can be easily tested by injecting a test context into the class instead of the real one

By default we'll use the name of the class as activity name (when communicating with the server).
This is of course overridable in case the SDK users would like to use their own naming schema. And
we'll use the method `#execute` as the primary entrypoint for executing user code.

```ruby
class MyActivity < Temporal::Activity
  ManuallyCancelled = Class.new(Temporal::Error)

  def execute(arg_1, arg_2: nil)
    loop do
      sleep 1
      activity.logger.info('❤️')
      activity.logger.debug("Attempt #{activity.info.attempts}")
      activity.heartbeat('details')
    end
  rescue Temporal::ActivityCancelled
    raise ManuallyCancelled, 'activity got cancelled'
  end
end
```

Notes:

- `activity` is an instance of `Temporal::Activity::Context`
- `activity.logger` will use a logging adapter configure via `Temporal::Configuration` (more on
  this in the 2nd phase)
- Common logging context can be configured by using middleware (more on this in the 2nd phase)
- `activity.heartbeat` is a synchronous call
- Errors defined within the activity class will be namespaces `MyActivity::ManuallyCancelled`,
  which is a very useful property

```ruby
class Temporal::Activity::Context
  # Access to a pre-configured logger
  def logger() -> Temporal::Logger

  # Heartbeat will raise if a cancellation was requested
  def heartbeat(*details) -> void

  # Provides access to extra details from PollActivityTaskQueueResponse
  def info() -> Temporal::Info::Activity
end
```

## Payload converters

TODO

