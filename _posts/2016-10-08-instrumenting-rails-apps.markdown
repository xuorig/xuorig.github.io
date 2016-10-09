---
title:  "Instrumenting Rails apps without impacting performance: Behind New Relic's Ruby Client"
date:   2016-10-08 12:00:00
description: "My deep dive notes on RPM: New Relic's Ruby Client"
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="Instrumenting Rails apps without impacting performance: Behind New Relic's Ruby Client">
<meta name="twitter:description" content="Instrumenting Rails apps without impacting performance: Behind New Relic's Ruby Client">
<meta name="twitter:image" content="">

#### *Surprise! A blog post that is not about GraphQL ;)*

Actually, it kind of begins with GraphQL. I was recently looking into ways of instrumenting GraphQL execution. [graphql-ruby][gem] is about to get instrumentation events: [https://github.com/rmosolgo/graphql-ruby/issues/186][issue] and knowing exactly what time our resolvers take to resolve / logging deprecated field usage is really interesting.

At the same time, Apollo is also releasing something called [Apollo Optics][optics]. It looks great and it would be great to send data to Optics from our Rails app. At the moment, there isn't really any finished client for Optics so I looked at how we could implement one in Ruby.

The main blocker here is that we're using Rails with [*Unicorn*][uni]. Unicorn is a pre-forking webserver and each request Unicorn handles is handled by one process, which is blocked until the request is finished. Sending data as an inline call to Optics or any other server during a request would be sad; we would block the process longer and also delay the response for the client. It shouldn't impact our response time!

I'll admit I  was a bit confused on how to solve that problem, but I also knew that it was possible since it's been done many times before! [@cjoudrey][cjoud] pointed me to [RPM][rpm], the New Relic Ruby client, which can be used easily in a Rails app to instrument a whole lot of different metrics.

I explored the code base for some (a lot of) time and  got out alive. Here's a few notes on how they make it work without impacting your app's performance!


# RPM

Repo: [https://github.com/newrelic/rpm][rpm]

Disclaimer: This post won't actually be about how to use [RPM][rpm] in your app. It's actually quite simple, the README is right here: https://github.com/newrelic/rpm. Instead we'll focus on how things are implemented inside!

Disclaimer 2: RPM works with pretty much all popular Ruby webservers, and also a lot of popular Ruby Web Frameworks. In this post we'll focus on how they make it work for Unicorn & Rails. Alright let's start!

Let's go back to the original problem. We are using Unicorn, a pre-forking web server to handle requests. Making http calls to the New Relic server is quite expensive and we can't afford to do that inline. As some of you maybe already guessed, RPM uses an "Agent Thread" to make those calls.

The first thing that surprised me is that the Agent Thread isn't started when you boot the App when you're using Unicorn or another pre-forking server:

{% highlight ruby %}
def start_agent
  return if using_forking_dispatcher?
  setup_and_start_agent
end
{% endhighlight %}

{% highlight ruby %}
# If we're using a dispatcher that forks before serving
# requests, we need to wait until the children are forked
# before connecting, otherwise the parent process sends useless data
def using_forking_dispatcher?
  if [:unicorn, ...].include? Agent.config[:dispatcher]
    true
  else
    false
  end
end
{% endhighlight %}

You see here that if we're using unicorn, RPM won't actually start the Agent thread when asked, and will instead wait until later. When ? When we actually receive a request to be traced! Take a look at this method inside the `NewRelic::Agent::Harvester` class. The job of that class is to wait for a new transaction (request) and to start our Agent Thread when we receive it!

*I believe the reason why RPM doesnt start the agent on boot is that since Unicorn is a pre-forking server, we want to make sure the Agent is ran inside the actual processes. Waiting for a transaction to start makes sure we'll be inside one of those processes!*

{% highlight ruby %}
  def on_transaction(*_)
    restart_harvest_thread
  end
{% endhighlight %}

What `Harvester#restart_harvest_thread` ends up calling is `Agent#start_worker_thread`. `start_worker_thread` spawns a new Thread using their own abstraction `NewRelic::Agent::Threading::AgentThread`, and that thread starts working!

### The Agent Thread

Inside that thread is where things start getting really interesting. Here's what is ran inside:

{% highlight ruby %}
def create_and_run_event_loop
  @event_loop = create_event_loop
  @event_loop.on(:report_data) do
	transmit_data
  end

  @event_loop.fire_every(Agent.config[:data_report_period], :report_data)

  @event_loop.run
end
{% endhighlight %}

Wow! What's that event loop and what the hell is it doing ? We can see we seem to be subscribing to events and providing callbacks (transmit_data). And it also seems like we're creating some sort of timer with `fire_every`.

`create_event_loop` gives us an instance of `EventLoop`. If we take a look inside the `NewRelic::Agent::EventLoop` class, we discover a pattern similar to what some of you might already be familiar with: [The Reactor Pattern][reactor].

### The EventLoop

{% highlight ruby %}
# NewRelic::Agent::EventLoop

def run_once
  # wait until we have stuff to do before executing once ( more on this later )
  wait_to_run

  # Execute timers which will populate the event_queue
  fire_timers

  # Consume all events that we have received
  until @event_queue.empty?
    evt, args = @event_queue.pop
    dispatch_event(evt, args)
    reschedule_timer_for_event(evt)
  end
end


{% endhighlight %}

As you can see, the main loop takes care of firing any timers we've added, and also reads anything we might've put inside it's event queue. That loop repeats until we instruct it to stop. You might've noticed the `wait_to_run` method called on top of the loop. Take a look at the implementation:

{% highlight ruby %}
# NewRelic::Agent::EventLoop

def wait_to_run
  # next timeout calculates how much time left until next timeout
  timeout = next_timeout

  # IO.select will block the loop until its time to execute a timer or
  # if we have received something to read in @self_pipe_rd (more on that later)
  ready = IO.select([@self_pipe_rd], nil, nil, timeout)

  if ready && ready[0] && ready[0][0] && ready[0][0] == @self_pipe_rd
    @self_pipe_rd.read(1)
  end
end
{% endhighlight %}

The `wait_to_run` method blocks the execution using `IO.select`. It only blocks for the amount specified by `timeout`. In this case the timeout is the amount of time left until we need to fire the next timer! But what if theres a lot of time left before the next timer needs to be fired and we need to add an event in the queue ? Since it's stuck in the loop waiting for the timer, our EventLoop won't be able to execute the callback!

This is why you see `@self_pipe_rd`. When we fire an event using the `fire` method, we'll write a byte into `@self_pipe_rd` to "wakeup" the event loop, skipping the specified timeout using `IO.select`. This is the good ol' "self-pipe" trick! Read more about it here: [https://www.sitepoint.com/the-self-pipe-trick-explained/][selfpipe]

OK, remember when we added timers to our event loop: `@event_loop.fire_every(Agent.config[:data_report_period], :report_data)`. With that we added a timer to the loop, and when it's time to fire that timer, the event loop will queue the `:report_data` in the `event_queue` once the loop pops that event from the queue it will execute its callback:

{% highlight ruby %}
  @event_loop.on(:report_data) do
    transmit_data
  end
{% endhighlight %}

Finally! We're sending our instrumentation data to New Relic's servers. But where did we get that data from, and how do we access it ?

{% highlight ruby %}
def transmit_data
  @service.session do
    harvest_and_send_errors
    harvest_and_send_error_event_data
    harvest_and_send_transaction_traces
    harvest_and_send_slowest_sql
    harvest_and_send_timeslice_data

    check_for_and_handle_agent_commands
    harvest_and_send_for_agent_commands
  end
end
{% endhighlight %}

As you see, we're sending a lot of different data, but for this post I will focus only on errors. All different types of data are handled in a similar way.

`@service.session` will try to open a keep-alive connection with new relics servers so we can send it all these requests inside a single connection.

As I just said, Lets focus on `harvest_and_send_errors`. All these `harvest_*` methods are quite similar. They call `harvest_and_send_from_container` specifying which 'container' they want to harvest from, and which type of data it wants from it.

"Containers" are an abstraction in RPM. All containers define an `harvest!` method, which will return an array of data to be sent.

In our case, the container is an instance of `NewRelic::Agent::ErrorTraceAggregator`. If we look at the source, we indeed see the `harvest!` method:

{% highlight ruby %}
  # Get the errors currently queued up.  Unsent errors are left
  # over from a previous unsuccessful attempt to send them to the server.

  def harvest!
    @lock.synchronize do
      errors = @errors
      @errors = []
      errors
    end
  end
{% endhighlight %}

The `harvest!` method's job is quite simple, it takes all the errors it aggregated and returns them, while reseting it's own aggregator array to an empty array `[]`.

Notice we used `@lock`, a `Mutex` to synchronize that behaviour. This is because we're inside the Threading Agent, but our Unicorn Process / Rails App might be also writting to it at the same time!


Speaking of which, how did data end up in that aggregator ? It's time to leave the `Agent` and take a loop how RPM integrates with Rails to populate these agregators.

### Instrumenting the Rails Controller

RPM needs a way to hook into our normal Rails controller so it can trace it's behaviour. The way they did it is by reimplementing `process_action` in a module. That module is included by RPM inside `ActionController` by default if you use the gem.

{% highlight ruby %}
def process_action(*args) #THREAD_LOCAL_ACCESS
  perform_action_with_newrelic_trace(
    :category => :controller,
    :name => self.action_name,
    :path => newrelic_metric_path,
    :params => munged_params,
    :class_name => self.class.name)  do
      super
    end
  en
end
{% endhighlight %}

RPM is basically wrapping `process_action` with it's own instrumentation method, allowing it to catch errors, time the response time, and anything you can imagine!

Inside `peform_action_with_newrelic_trace` we find the Core class in charge of instrumenting our app: `NewRelic::Agent::Transaction`

The result is that `process_action` is wrapped between a `Transaction.start` statement and a `Transaction.stop` statement.

Remember the `on_transaction` method that made our agent boot up ? Well `Transaction.start` dispatches that event!

There is a __lot__ of stuff happening inside the `Transaction` class. I'll stay the errors example and take a look at the `Transaction.stop` part, where errors are instrumented. `#stop` records metrics by finally calling the `commit!` method. This is where RPM fills our agregators / containers with data.

Here's a simplified version of the `commit!` method:

{% highlight ruby %}
def commit!(state, end_time, outermost_node_name)
  # ... record a bunch of things
  record_exceptions
  record_transaction_event
  send_transaction_finished_event
end
{% endhighlight %}


What do you think `record_exceptions` does ? Queues up an error in our error_aggregator!

{% highlight ruby %}
def record_exceptions
  @exceptions.each do |exception, options|
    options[:uri]      ||= request_path if request_path
    options[:port]       = request_port if request_port
    options[:metric]     = best_name
    options[:attributes] = @attributes

    # Add errors to the aggregator
    agent.error_collector.notice_error(exception, options)
  end
end
{% endhighlight %}

With this finished we have the most important pieces of RPM's instrumentation. Let's make a little summary of our findings.

### TL;DR

  - RPM monkey patches / wraps Rails with it's own instrumentation methods
  - These methods, instead of sending the data immediatly, queue up the data in "containers" / aggregators.
  - These aggretator are shared objects between our Rails App and the Agent Thread.
  - When RPM begins a transaction on unicorn, it boots up the Agent Thread which, using a clever EventLoop, triggers the transmition of data.
  - The Agent pops off data from the aggregators, and sends it over an http long lived connection.

Of course I haven't covered everything. There's a lot of classes for different aggregators and there's also a lot of complex behaviour inside the `Transaction` class.

Diving into RPM gave me a lot of ideas on how to implement my own AgentThread for GraphQL instrumentation. I really hope it gave you some ideas!

If I missed anything or said blatantly wrong thing please let me know :)

As always, you can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[gem]: https://github.com/rmosolgo/graphql-ruby
[issue]: https://github.com/rmosolgo/graphql-ruby/issues/186
[optics]: http://www.apollostack.com/optics
[cjoud]: https://twitter.com/cjoudrey
[uni]: https://github.com/defunkt/unicorn
[rpm]: https://github.com/newrelic/rpm
[selfpipe]: https://www.sitepoint.com/the-self-pipe-trick-explained/
[reactor]: https://en.wikipedia.org/wiki/Reactor_pattern
