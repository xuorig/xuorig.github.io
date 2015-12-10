---
title:  "Uploading files using Relay and a Rails GraphQL server"
date:   2015-12-10 10:18:00
description: How to upload a file using graphql-ruby's context and Relay's getFiles
---

I recently needed to upload a file from a [React-Relay][relay] App to my [GraphQL][graphql] Rails server
and found out it was not as straight forward as I thought and not very documented. Thought I'd
share what I have done to make file uploading work with Rails and Relay. Big thanks to
[Robert Mosolgo][mosolgo] who guided me towards this solution.

I you haven't checked out my first blog about [GraphQL][graphql] you can see it [here][blog].

# Relay

Relay Mutation have a method called __getFiles__ which allows you to send files
to a server. It should return a map of files like this __{[key: string]: File}__
Here's a quick example of how we could use it:

{% highlight javascript %}
// A react component to upload a file
class FileUploader extends React.Component {
  onSubmit() {
    const name = this.refs.name.value;
    const file = this.refs.fileInput.files.item(0);
    Relay.Store.update(
      new UploadFileMutation({
        name: name,
        file: file,
      })
    );
  }
}

// The actual mutation
class UploadFileMutation extends Relay.Mutation {
  getFiles() {
    return {
      file: this.props.file,
    };
  }

  // ... Rest of your mutation
}
{% endhighlight %}


# GraphQL Rails API

The part that gets more confusing is how to receive that file server side. If
you're using GraphQL-js there is this [example][example] that explains how you
would design your schema to receive files.

Unfortunatly this wasn't really possible using [graphql-ruby][graphruby] and I had
to find another way to do it. Luckily, [graphql-ruby][graphruby] passes a context argument to
the nodes when it resolves a query. That means we can pass the files along to the query
in the controller and retrieve it at resolve time in the mutation.

Here's how you could write your controller which in my case is called __QueriesController__.

{% highlight ruby %}
module V1
  class QueriesController < ApplicationController

    def create
      query_string = params[:query]
      query_variables = ensure_hash(params[:variables]) || {}

      query = GraphQL::Query.new(
        YourSchema,
        query_string,
        variables: query_variables,
        context: { file: request.params[:file] }
      )

      render json: query.result
    end

    # If the request wasn't `Content-Type: application/json`, parse the variables:
    def ensure_hash(variables_param)
      variables_param.kind_of?(Hash) ? variables_param : JSON.parse(variables_param)
    end
  end
end
{% endhighlight %}

You'll notice that there is an __ensure_hash__ method added here. Variables in Relay mutations
with files aren't sent with a `Content-Type: application/json` so they arrive as
a String in your controller. Here we simply check if the variables are sent as a Hash and if
they're not we parse them as JSON.

The rest is pretty straight forward, we add the file param to our context so it can be passed
along during resolve. That means that our Mutation on the server side can look a little bit
like this:

{% highlight ruby %}
AddFileMutation = GraphQL::Relay::Mutation.define do
  name "AddFile"
  input_field :name, !types.String

  # ... Add your return_field here

  # Here's the mutation operation:
  resolve -> (args, ctx) {
    file = ctx[:file]
    raise StandardError.new("Expected a file") unless file
    # ... Do what you want with the file!
  }
end
{% endhighlight %}

Here you have it! Note that *you should probably not be uploading large files on
the server* and you might want to upload it browser side!

Hope it'll help some of you!



[relay]: https://github.com/facebook/relay
[graphql]: https://github.com/graphql
[mosolgo]: https://twitter.com/rmosolgo
[blog]: http://mgiroux.me/2015/getting-started-with-rails-graphql-relay/
[example]: https://github.com/graphql/express-graphql/blob/master/src/__tests__/http-test.js#L603
[graphruby]: https://github.com/rmosolgo/graphql-ruby
