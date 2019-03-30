In addition to fetching data using queries and modifying data using mutations, 
the GraphQL spec supports a third operation type, called `subscription`.

GraphQL subscriptions are a way to push data from the server to the clients that choose to listen to real time messages 
from the server. Subscriptions are similar to queries in that they specify a set of fields to be delivered to the client, 
but instead of immediately returning a single answer, a result is sent every time a particular event happens on the server.

A common use case for subscriptions is notifying the client side about particular events, 
for example the creation of a new object, updated fields and so on.

>> GraphQLBundle since `v1.3` support subscriptions but this feature
is still in experimental state.

# When to use subscriptions

In most cases, intermittent polling or manual refetching are the best way to keep your client up to date. 
So when is a subscription the best option? Subscriptions are especially useful if:

### Live Availability
- A webapp retrieves the availability status of a product from a API and displays it: only one is still available
3 minutes later, the last product is bought by another customer
the webapp's view instantly shows that this product isn't available anymore

### Asynchronous Jobs
- A webapp tells the server to compute a report, this task is costly and will take some time to complete
the server delegates the computation of the report to an asynchronous worker (using message queue), and closes the connection with the webapp
the worker sends the report to the webapp when it is computed

### Collaborative Editing
- A webapp allows several users to edit the same document concurrently and changes made are immediately broadcasted to all connected users

# How it's Works?

GraphQLBundle has a custom integration of subscriptions using mercure protocol 
and following [graphql spec guidelines](https://graphql.github.io/graphql-spec/draft/#sec-Subscription).

The internal workflow is as follow:

- The user send a subscription query to the server
- The server save the request as is and link that to a new subscription ID
- The server mark this subscription as non used and set the ttl in ~20s
- The subscription url is returned to the client
- The client create a `EventSource` with given url
- The server mark this subscription has used and increase the ttl (default: 5minutes)
- Some time later...
- The server dispatch a subscription update
- The server find all subscriptions matching given filters
- The server get the previously saved request and re-send it
- The server process the request and dispatch a update to mercure hub with the given response