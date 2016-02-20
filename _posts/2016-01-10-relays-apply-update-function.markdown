---
title:  "Relay's new applyUpdate function"
date:   2016-01-10 17:00:00
description: How to use Relay's applyUpdate function and retry a failed transaction
---

I recently contributed to Relay and helped implementing a new function on the Relay Store,
`applyUpdate`. It was recently merged into master and the standard `update` method was deprecated in ```v0.6.1```.

Before this, we were limited to `RelayStore.update` when wanting to use Mutations. For those who are less familiar with how relay mutations work under the cover, Relay has an internal queue to handle mutations. When using `RelayStore.update`, we were essentially creating a mutation transaction in the queue and committing it instantly.

This actually worked well in most cases, but for more advanced use cases like for debouncing quick mutations on a single field, we needed something else. This is where `RelayStore.applyUpdate` gets useful. The `applyUpdate` function adds a mutation just like `update`, but does not commit it. It returns a `RelayMutationTransaction` that can be committed, recommitted or rollbacked at a later time.

# Example use case

I've recently seen somebody ask this question on StackOverFlow: We want to enable
retrying a failed mutation. Lets take an example from [Relay's docs][docs] to do this.

So we want to post a Story, but if it fails, we want to display a link to retry posting
the same story. To do this using `applyUpdate`, we could keep for example the transaction
in our Component state. When we try the mutation, we create it with applyUpdate, store it,
and try to commit it after.

We now have a reference to this transaction, if we see that the transaction has
a status of COMMIT_FAILED during render, we can actually show the the link to retry,
and call `transaction.recommit` to try again. Using only `update` we would lose our
reference to the transaction and doing more things with the transaction after becomes
much more complicated. Here's what your component could look like:

{% highlight javascript %}
class Story extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      transaction: null
    };
  }

  postStory(transaction) {
    const onSuccess = () => {
      console.log('Mutation successful!');
    };

    const onFailure = (transaction) => {
      const error = transaction.getError()
      console.error(error);
    };

    const mutation = new MyMutation({...});
    let transaction = Relay.Store.applyUpdate(
      mutation, { onFailure, onSuccess}
    );

    this.setState({transaction: transaction});
    transaction.commit();
  }

  retryPostStory() {
    let transaction = this.state.transaction;
    transaction.recommit;
  }

  render() {
    var story = this.props.story;
    let transaction = this.state.transaction;

    if (transaction) {
      if (transaction.getStatus() === 'COMMIT_FAILED') {
        return (
          <span>
            This story failed to post.
            <a onClick={this.retryPostStory.bind(this)}>
              Try Again.
            </a>
          </span>
        );
      }
    }
    // Render story normally.
  }
}

module.exports = Relay.createContainer(Story, {
  fragments: {
    story: () => Relay.QL`
      fragment on story {
        # ...
      }
    `,
  },
});
{% endhighlight %}

I have seen a lot of questions on using Relay that could be solved using apply_update really easily.
Also keep in mind that `update` is now deprecated and you should use `RelayStore#commitUpdate` to get the same behavior!

You can find me on Twitter [@\_\_xuorig\_\_][twit] or [Github][xuo]

[twit]: https://twitter.com/__xuorig__
[xuo]: http://github.com/xuorig
[app]: https://github.com/xuorig/my-simple-blogging-app
[docs]: https://facebook.github.io/relay/docs/api-reference-relay-container.html#getpendingtransactions
