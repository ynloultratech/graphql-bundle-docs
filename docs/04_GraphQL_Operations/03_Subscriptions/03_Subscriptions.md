Subscriptions works similar to [Queries](../01_Queries.md) and [Mutations](../02_Mutations.md), 
to define a Subscription must use the `@GraphQL\Subscription()` annotation
and create the [resolver](../../08_Reference/03_Resolvers.md) using [naming convention](../../08_Reference/02_Naming_Conventions.md).

Subscription Resolvers:
 
 `Subscription\{Node}\{OperationName}`
 
Then, if you need create subscription for `User` node:

````php
namespace App\Subscription\User;

/**
 * @GraphQL\Subscription()
 */
class OnAddUser
{
    public function __invoke()
    {
       ...
    }
}
````

Options:
- **name**: Name to expose the subscription, if not set will be automatically resolved.
- **description**: Subscription description to expose in the documentation
- **type**: The return type of the subscription, can use NonNull and List modifiers
- **fieldName**: Name of the field to use in the event object, default `data`
- **fieldDescription**: Description to expose in the field
- **deprecationReason**: Mark the field as deprecated with the following reason
- **options**: Options are used by [plugins](../../07_Advanced/99_Definitions_Plugins.md) to provide extra features
 
# Subscription Response

The response of each subscription should match with the exposed type in the GraphQL schema. 
If your subscription expose a `User` type as subscription result then must return this type of object in the return statement in the resolver.

````
namespace App\Subscription\User;

use Ynlo\GraphQLBundle\Annotation as GraphQL;
use App\Entity\User;

/**
 * @GraphQL\Subscription()
 * @GraphQL\Argument(name="username", type="String!")
 */
class OnAddUser
{
    protected $em;

    public function __construct(EntityManagerInterface $em)
    {
        $this->em = $em;
    }
    
    public function __invoke($userId)
    {
        return $this->em->getRepository(User::class)->find($userId);
    }
}
````

# Publishing

In order to push a subscription update must inject the `publisher` service where you need it.

````php
use Symfony\Component\HttpFoundation\Response;
use Ynlo\GraphQLBundle\Subscription\Publisher;
use App\Subscription\User\OnAddUser;

class UserManager
{
    /**
     * @var Publisher
     */
    protected $publisher;

    /**
     * @param Publisher $publisher
     */
    public function __construct(Publisher $publisher)
    {
        $this->publisher = $publisher;
    }
    
    public function createUser(): Response
    {
        // the logic to create the user and persist in database

        $this->publisher->publish(OnAddUser::class, [], ['userId'=> $user->getId()]);
    }
}
````

The publish method of the `publisher` object allow 3 arguments:

- **name**: Subscription class
- **filters**: Subscription filters to apply
- **data**: Subscription arguments to send to the subscription

In the above example when a new user is created the subscription `OnAddUser` is called
with `userId` as argument. Then in the subscription can return the real user.

````php
public function __invoke($userId)
{
    return $this->em->getRepository(User::class)->find($userId);
}
````

> The third argument in the `publish` method is a array of arguments to pass to the subscription 
__invoke() method and each array key should be equal to the argument name.

>> The array of arguments to pass must be a array of scalar values, objects are not allowed.

# Execution, how it works?

Subscriptions are executed asynchronously, the first time a subscription is executed the API consumers
receive a `SubscriptionLink` object and then receive a event object every time the subscription is dispatched.

The response of each subscription is a union and may contain one of the following types:

- `SubscriptionLink` Received when a subscription is executed
- `SubscriptionSpecificEvent` Received every time a subscription is dispatched.

GraphQLBundle save the original client request to emulate that request every time a subscription is dispatched.
With this approach you inclusive can verify for user permissions during the execution.

>> Subscriptions are executed with the same endpoint, credentials etc. used to create them. If the API
use some authentication mechanism like JWT, the token used to subscribe will be used to receive 
subscriptions updates, ensure your client know this in order to re-subscribe when receive a invalid authentication
response during a subscription update.

# SubscriptionLink

A `SubscriptionLink` object is received by API consumers when a 
subscription is called directly and contains the following properties:

- `url` corresponding subscription url containing a unique subscription ID. The client can
        subscribe to the event stream corresponding to this subscription by creating a `new EventSource`.
- `ttl` Time to live(in seconds) for a subscription, must call a heartbeat to keep the subscription alive.
- `heartbeatUrl` Url to send periodicals requests (heartbeats) to keep-alive the subscription.

Once the consumer receive this link is ready to subscribe to the [event stream](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).

> Every time a subscription is created is available to subscribe in a limited time(~20 seconds),
for that reason the `url` in the link is a middleware url to the real Mercure hub. Internally the main target of this approach
is mark the subscription as used and increase the ttl to configured ttl *(default: 5 minutes)*.

### Why the `ttl` and `heartbeatUrl`?

Unlike other subscriptions systems, GraphQL must store each subscription individually in order to send
to each subscriber only requested fields. Because the event source server is managed by Mercure your API
does not have any way to know if some subscription should be still active. 

In order to keep the server clean of non used subscriptions all subscriptions have a time to live *(default: 5 minutes)*, 
after this time the subscription will be removed in the server and not dispatched any more. 

Client must send a heartbeat to the server periodically to the `heartbeatUrl` in order to
mark the subscription as still active.

> The heartbeat request is a `GET` request to the `heartbeatUrl`, the response of the server 
should be `200` as status code with empty response.
The heartbeat interval must be any time lowest than subscription ttl.

Can configure subscriptions `ttl` changing the following configuration,
but it is highly recommended does not use values that are too high.

````yaml
graphql:
    # Manage subscriptions settings
    subscriptions:
        # Time to live for subscriptions. The subscription will be deleted after this time, a heartbeat is required to keep-alive
        ttl:  300

````

# Subscription Event

Every time a subscription is dispatched subscribers receive a Subscription Event object. 
This object only contains only one property *(default: data)* and the type of this property 
is the same specified in the subscription payload.

````
namespace App\Subscription\User;

use Ynlo\GraphQLBundle\Annotation as GraphQL;
use App\Entity\User;

/**
 * @GraphQL\Subscription(payload="User", fieldName="user")
 */
class OnAddUser
{
    protected $em;

    public function __construct(EntityManagerInterface $em)
    {
        $this->em = $em;
    }
    
    public function __invoke($userId)
    {
        return $this->em->getRepository(User::class)->find($userId);
    }
}
````

> Can change the name of the field in the Event using the property `fieldName`.

The above subscription can be used with following query:

````graphql
subscription {
  onAddUser {
    ... on SubscriptionLink {
      url
      heartbeatUrl
      ttl
    }
    ... on OnAddUserEvent {
      user {
        id
        username
      }
    }
  }
}
````