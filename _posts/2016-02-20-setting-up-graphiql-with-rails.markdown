---
title:  "Setting up GraphiQL in a Rails Application"
date:   2016-02-20 12:00:00
description: Learn how to setup GraphiQL in your Rails Application and test your queries without problem
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="Setting up GraphiQL in a Rails Application">
<meta name="twitter:description" content=" Learn how to setup GraphiQL in your Rails Application and test your queries without problem">
<meta name="twitter:image" content="https://cloud.githubusercontent.com/assets/1919498/13198607/7ec37800-d7db-11e5-9d1b-55dcd982bd63.png">

A cool thing that GraphQL's Introspection brings us is the ability to make awesome tools around your schema. One of them is [GraphiQL][graphiql] (Graphical). GraphiQL enables you to test queries against you schema without the need of using curl or other tools. It has autocomplete which is really awesome, and a lot of other nice features.

## Installation

### With Gem

You can use the [GraphiQl-Rails][gem] gem to easily mount the Graphiql endpoint in your Rails app. The steps are clearly outlined in ReadMe.

### Manually

To use GraphiQL we'll need 4 different js files in our assets. Lets just add them in `app/assets/javascripts` but you could place them wherever you want.

#### Graphiql.js

You can get the graphiql source right [here][graphiql]. I personally used their `build.sh` script. Clone the repo and run this script. You should then have a `graphql.js` and a `graphql.css` in the root folder. Copy the js file into your javascripts and the css file in your stylesheets.

Note: I personally had to add this css rule for `graphiql` to render correctly
{% highlight css %}
html, body {
  height: 100%;
}
{% endhighlight %}

Next!

#### fetch.js

Graphiql needs some kind of library to call your GraphQL server. They recommend [fetch][fetch]. To get the `fetch.js` file, head to [Github's fetch polyfill repo][github] and download the `fetch.js` file. Place it in your assets too!

#### React and React-Dom

GraphiQL uses react components to render, so we'll need that too. Clone [React][react], and run `npm install`. If you haven't already, install `grunt` and `grunt-cli` globally using `npm`. Then, run `grunt build`.

You should now have a build folder containing, both `react.js` and `react-dom.js`. Add both of them to your assets!

Ok, now let's include all those assets in our `application.css` and `application.js` files.

#### application.css
{% highlight css %}
/*
 = require ./graphiql
 */
{% endhighlight %}

#### application.js
{% highlight js %}
//= require ./react
//= require ./react-dom
//= require ./fetch
//= require ./graphiql
{% endhighlight %}

We have finally all we need to start using graphiql. The first step is to create a controller that will render GraphiQL.

{% highlight ruby %}
# app/controllers/graphiql_controller.rb
class GraphiqlController < ApplicationController
  def show
  end
end
{% endhighlight %}

As you can see the controller does nothing special, our work will be mainly in the view, in `app/views/graphiql/show.html.erb`:

{% highlight html %}
<div id="graphiql-container"></div>

<script>
// Define your graphQLFetcher (Which will call your graphql api)
function graphQLFetcher(graphQLParams) {
  return fetch(window.location.origin + '/queries', {
    method: 'post',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(graphQLParams),
  }).then(response => response.json());
}
ReactDOM.render(
  React.createElement(GraphiQL, {
    fetcher: graphQLFetcher,
  }),
  document.getElementById("graphiql-container")
);
</script>
{% endhighlight %}

We render the `GraphiQL` React component, and pass it our fetcher, which will post to our `queries_controller`. (Replace this by wherever you execute GraphQL queries in your App.)

Finally, lets add a route so we can access GraphiQL. In `routes.rb` add this line:

{% highlight ruby %}
  get 'graphiql', to: 'graphiql#show'
{% endhighlight %}

Now head to `/graphiql` and you should see the GraphiQL interface! Time to test your queries :)

<img width="595" alt="screenshot 2016-02-20 14 06 19" src="https://cloud.githubusercontent.com/assets/1919498/13198607/7ec37800-d7db-11e5-9d1b-55dcd982bd63.png">

You can see an example of this setup at my [Simple Blogging App][app] repo.

You can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[graphiql]: https://github.com/graphql/graphiql
[gem]: https://github.com/rmosolgo/graphiql-rails
[fetch]: https://fetch.spec.whatwg.org/
[github]: https://github.com/github/fetch
[react]: https://github.com/facebook/react
[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[app]: https://github.com/xuorig/my-simple-blogging-app
