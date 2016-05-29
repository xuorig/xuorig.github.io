---
title:  "Real time updates with Relay and Rails' ActionCable"
date:   2016-05-29 10:00:00
description: Setting up live updates with Rails' ActionCable, GraphQL and Relay
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="Real time updates with Relay and Rails' ActionCable">
<meta name="twitter:description" content="Setting up live updates with Rails' ActionCable, GraphQL and Relay">
<meta name="twitter:image" content="http://mgiroux.me/assets/images/realtimerelay.gif">

To me, the only thing missing to the GraphQL ecosystem and spec today are live updates. When our client applications require live updates from the server for data they care about, we need some sort of mechanism and the GraphQL team has talked about introducing a third operation besides `query` and `mutation`: the [subscriptions][sub].

Unfortunatly for us, the spec hasn't described the "way" to do it just yet and no real implementation or spec of `subscriptions` exists. The good news is that it's still possible to have real time updates in our Relay applications using our current existing strategies, such as a standard WebSocket connection.

This weekend I took that challenge and at the same time gave Rails' `ActionCable` for a ride.

## A bit of context on what we'll build

I'm building [ReviewBuddy][reviewbud] on my spare time. ReviewBuddy is a small app to keep track of your own pull requests, and other pull requests that you need to review. Now, when somebody updates these pull requests, I'd like the pull requests on review buddy to be udpated too. It would be even better if it was in real time!

## ActionCable

Let's setup ActionCable first. Since my client app doesn't live inside my Rails GraphQL API, we'll need to whitelist that domain for ActionCable to work:

{% highlight ruby %}
  # config/environments/development.rb
  config.action_cable.allowed_request_origins = ['http://localhost:8080']
{% endhighlight %}

ActionCable works using "Channels". For anything we need to stream to client, we have to define a `Channel`. For PullRequests, I've created a `PullRequestsChannel`:

{% highlight ruby %}
# app/channels/pull_requests_channel.rb
class PullRequestsChannel < ApplicationCable::Channel
  def subscribed
    stream_from 'pull_requests'
  end
end
{% endhighlight %}

The only thing left to do is to actually stream data when something has changed! We could simply have an `after_commit` that streams the pull request when something changed, but that's always a bad desision ;). let's just broadcast from where we modify the pull request.

All the Pull Request updates happen in a job. On perform, we find the PullRequest from the Github webhook params, and we update what we care about.

After a successful save, we broadcast the changes to the client using `ActionCable.server.broadcast`, using the `pull_requests` stream and our `pull_request` as a param.

{% highlight ruby %}
  def perform(params)
    pr = PullRequest.find_by(
      repo: params[:repository][:name],
      owner: params[:repository][:owner][:login],
      number: params[:number],
    )
    return unless pr

    pr.title = params[:pull_request][:title]
    pr.merged = params[:pull_request][:merged]

    if pr.save!
      broadcast_pr_change(pr)
    end
  end

  def broadcast_pr_change(pr)
    ActionCable.server.broadcast 'pull_requests',
      pull_request: {
        id: pr.to_global_id.to_s,
        title: pr.title
      }
  end
{% endhighlight %}

## Relay Client App

Now the hard thing was communicating with `ActionCable` in the `Relay` app, and also updating the `Relay` Store. `ActionCable` is really tied to `Rails`, and trying to use it with an external client side app + API was harder than expected.

With the help of this [repo][repo], I was able to create own javascript client to implement the `ActionCable` protocol.

{% highlight javascript %}
export default class ActionCableClient {
  constructor(uri, channel, onMessage) {
    this.uri = uri;
    this.channel = channel;
    this.onMessage = onMessage;

    this.socket = new WebSocket(uri);
    this.socket.onmessage = this._handleMessage.bind(this);

    this._onReadyPromise = new Promise((resolve, reject) => {
      this._resolveOnReady = resolve;
      this._rejectOnReady= reject;
    });
  }

  static connect(uri, channel, onMessage) {
    const instance = new this(uri, channel, onMessage);
    return instance.connect();
  }

  connect() {
    this.socket.onopen = () => {
      this.socket.send(this._getSubscribePayload())
    };

    return this._onReadyPromise;
  }

  _handleMessage(event) {
    const msg = JSON.parse(event.data);
    if (msg.type === 'confirm_subscription') {
      this._resolveOnReady(this);
    } else {
      this.onMessage(msg);
    }
  }

  _getSubscribePayload() {
    return JSON.stringify({
      command: 'subscribe',
      identifier: JSON.stringify({
        channel: this.channel
      })
    });
  }
}
{% endhighlight %}

I've created a gist of that client right here [https://gist.github.com/xuorig/b167b27f75c149b8da5837b79585169f][gist].

And this is how it can be used inside your app:

{% highlight javascript %}
const myEventHandlerFunction = (event) => {
  console.log(event.message);
}

ActionCableClient.connect(
  'ws://localhost:3000/cable',
  'YourChannel',
  myEventHandlerFunction
).then(client => {
  client is guaranteed to be initialized properly
  when promise is resolved.
});
{% endhighlight %}

With the client finished, we have all we need to start speaking with our API in real time. The only challenge left is how to update the RelayStore from another source. This is where having an actual subscription would be amazing but for now we have to choose another path.

I used this amazing issue from the Relay repo https://github.com/facebook/relay/issues/541 to derive a solution to this problem. Turns out we can use the `Relay.Store.getStoreData().handleQueryPayload` to programatically apply a payload to the store.

Here's my `ActionCable` message handler

{% highlight javascript %}
onActionCableMessage(event) {
  const pull_request = event.message.pull_request;

  const query = Relay.createQuery(Relay.QL `
    query {
      node(id: $id) {
        ... on PullRequest {
          title
        }
      }
    }
  `, { id: pull_request.id });

  const payload = {
    node: {
      title: pull_request.title
    }
  };

  Relay.Store.getStoreData().handleQueryPayload(query, payload);
}
{% endhighlight %}

Basically, we build a query and it's respective payload, and make relay update it's store using `handleQueryPayload`. Here we simply update the pull request title for the example. Take a look at the result:

<img src="/assets/images/realtimerelay.gif"/>

That's it! We have real time updates in our Relay App using Rails ActionCable :)

Let me know if you've found other ways of doing it or if you have any problems following the post!

As always, you can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[sub]: http://graphql.org/blog/subscriptions-in-graphql-and-relay/
[reviewbud]: https://github.com/reviewbuddy/review-buddy
[repo]: https://github.com/NullVoxPopuli/action_cable_client#the-action-cable-protocol
[gist]: https://gist.github.com/xuorig/b167b27f75c149b8da5837b79585169f

