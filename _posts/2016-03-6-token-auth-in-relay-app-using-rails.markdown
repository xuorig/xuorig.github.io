---
title:  "Authentication in Relay Applications using Rails and GraphQL"
date:   2016-03-06 10:00:00
description: Using an Authorization Token to enable authentification in Relay applications using Rails with Devise and GraphQL
---

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:site" content="@__xuorig__">
<meta name="twitter:creator" content="@__xuorig__">
<meta name="twitter:title" content="Authentication in Relay Applications using Rails and GraphQL">
<meta name="twitter:description" content="Using an Authorization Token to enable authentification in Relay applications using Rails with Devise and GraphQL">
<meta name="twitter:image" content="https://cloud.githubusercontent.com/assets/2231765/9094460/cb43861e-3b66-11e5-9fbf-71066ff3ab13.png">

Most of apps today have users and require some form of Auth. It can be tricky to implement it in a Relay app because we don't have explicit control on what we send to the server.

Here's the way I've been doing it in my project, this solution assumes that you're using [Devise][devise] for authentication in your Rails app.

### Sending the token client side

The way I've implemented Authentication in my Relay app is by using an Authorization Token in the request headers.

First thing we're going to do is modify the `DefaultNetworkLayer` to always send the `Authorization` header when we have it. We're going to hold the token in `localStorage`.

{% highlight javascript %}
const token = localStorage.getItem('auth_token');
const headers = { Authorization: auth_token }

Relay.injectNetworkLayer(
  new Relay.DefaultNetworkLayer('http://localhost:3000/your_graphql_endpoint', {
    headers: headers
  })
);
{% endhighlight %}

Now everytime Relay fetches data, our token will be sent as a header so we can auth server side.

### Handling the token server side

#### Generating the token

If we want to use an `auth_token` to identify our users, we'll need that in our database. Let's create a migration to add the column:

{% highlight ruby %}
class AddAccessTokenToUser < ActiveRecord::Migration
  def change
    add_column :users, :access_token, :string
  end
end
{% endhighlight %}

Alright, now that we have that column, we can actually use it. We want every user to have an `auth_token`. What we can do is generate one everytime we create a new user, like this:

{% highlight ruby %}
  # models/user.rb
  after_create :update_access_token!

  def update_access_token!
    access_token = "#{self.id}:#{Devise.friendly_token}"
    save!
  end
{% endhighlight %}

A note here on how we generate that token. I've taken this idea from a great gist made by Jose Valim: [https://gist.github.com/josevalim/fb706b1e933ef01e4fb6][https://gist.github.com/josevalim/fb706b1e933ef01e4fb6]

We could've used only `Devise.friendly_token` for the token, and then find the user by this token everytime we want to retrieve it, but it's not ideal for two main reasons: First, using `find_by(access_token: token)` is way slower that using the primary key like what we're going to do here. Second, searching for the token itself is vulnerable to timing attacks. Here is a great blog post about that type of attack: [https://codahale.com/a-lesson-in-timing-attacks/][https://codahale.com/a-lesson-in-timing-attacks/]

Keep in mind that using the id here is faster and avoids timing attacks but I still would not use that in production. You should probably encrypt the id instead of having that plain text in our token, since it's very obvious what it is in that case.

#### Authenticate using the token

The plan here is to be able to do `before_action :authenticate_user_from_token!` in our `graphql_controller`.

So lets define this method:
{% highlight ruby %}
  def authenticate_user_from_token!
    auth_token = request.headers['Authorization']
    return authentication_error unless auth_token
    authenticate_with_auth_token auth_token
  end
{% endhighlight %}

Fairly straight forward. We grab the token from the headers and return a `401` if it is not there.
If it is there, we're going to call `authenticate_with_auth_token` with it.
{% highlight ruby %}
  def authenticate_with_auth_token(auth_token)
    # return if the token does not have the right format
    return authentication_error unless auth_token.include?(':')

    # Find the user by splitting the token and finding by id
    # Again, not the most secure way to do it, but works as an example.
    user_id = auth_token.split(':').first
    user = User.where(id: user_id).first

    # Use secure_compare to mitigate timing atttacks
    if user && Devise.secure_compare(user.access_token, auth_token)
      sign_in user, store: false
    else
      authentication_error
    end
  end
{% endhighlight %}

We should be all set to start authenticating our users! Let's see how we can use this.
Here's an example of how you could handle graphql queries using the `graphql-ruby` gem.

{% highlight ruby %}
  # controllers/graphql_controller.rb
  before_action :authenticate_user_from_token!

  def create
    query_string = params[:query]
    query_variables = params[:variables] || {}
    result = YourSchema.execute(
      query_string,
      variables: query_variables,
      context: {current_user: current_user}, # Add the current_user into the query context
    )
    render json: result
  end
{% endhighlight %}

`authenticate_user_from_token!` sets our current_user correctly if the token matches
a user, or will fail with a `401` if it cannot.

We can now use current_user anywhere in the query execution. We can simply have a `currentUser`
field on our `QueryRoot`, that our Relay app fetches.

{% highlight ruby %}
  field :currentUser do
    type UserType
    resolve -> (obj, args, ctx) {
      ctx[:current_user]
    }
  end
{% endhighlight %}

You can then easily access the `currentUser` in your Relay app by including it in your fragments.

#### Signing in users

Now that we can authenticate users with their auth_token, it would be fun to actually get that `auth_token` when we don't have it in the first place, aka log in a user.

Turns out we can actually use a Relay Mutation for that purpose. Take a look at this mutation definition:

{% highlight ruby %}
SignInMutation = GraphQL::Relay::Mutation.define do
  name "SignIn"
  input_field :email, !types.String
  input_field :password, !types.String

  return_field :access_token, types.String

  resolve -> (args, ctx) {
    @user = User.find_for_database_authentication(email: args[:email])
    access_token = if @user.valid_password?(args[:password])
      @user.access_token
    end
    { access_token: access_token }
  }
end
{% endhighlight %}

The mutation is pretty straight forward too. We find the user by his email and validate the password passed in the arguments. If we successfuly authenticate the user, we can return his `access_token`!

On the Relay side, simply call the mutation with the `email` and `password` values coming from your sign in form, and set your local storage key with the `access_token` from the response.

Calling the mutation:
{% highlight javascript %}
  let onFailure = (transaction) => {
    var error = transaction.getError() || new Error('Mutation failed.');
    console.error(error);
  };

  let onSuccess = (response) => {
    const access_token = response.signin.access_token;
    localStorage.setItem('access_token', access_token);
    window.location = '/';
  }

  Relay.Store.update(new SignInMutation({
    email: email,
    password: password
  }), {onFailure, onSuccess});
{% endhighlight %}

The mutation itself (Notice the `REQUIRED_CHILDREN` config, which lets you force fragments to be included in the query):
{% highlight javascript %}
export default class SignInMutation extends Relay.Mutation {
  getMutation() {
    return Relay.QL`mutation {
      signin
    }`;
  }

  getVariables() {
    return {
      email: this.props.email,
      password: this.props.password,
    };
  }

  getFatQuery() {
    return Relay.QL`
      fragment on SignInPayload {
        access_token
      }
    `;
  }

  getConfigs() {
      return [{
        type: 'REQUIRED_CHILDREN',
        // Forces these fragments to be included in the query
        children: [Relay.QL`
          fragment on SignInPayload {
            access_token
          }
        `],
      }];
    }
}
{% endhighlight %}

If the mutation is a success, and your `access_token` is successfuly set in localStorage, your next queries will have the correct `Authorization` header, making the currentUser available in your queries!

Hope this will help some of you, I've seen a lot of different ways of handling Authentication in GraphQL/Relay apps, so let me know if you have any other ways of doing it!

You can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[https://codahale.com/a-lesson-in-timing-attacks/]: https://codahale.com/a-lesson-in-timing-attacks/
[https://gist.github.com/josevalim/fb706b1e933ef01e4fb6]: https://gist.github.com/josevalim/fb706b1e933ef01e4fb6
[devise]: https://github.com/plataformatec/devise

