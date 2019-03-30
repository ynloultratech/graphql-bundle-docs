Like [Queries](../01_Queries.md) and [Mutations](../02_Mutations.md), 
Subscriptions allow arguments to use as filters in order
to receive a subscription only if these arguments match with the filters applied
when the subscription is dispatched.

````php

namespace App\Subscription\User;

use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Ynlo\GraphQLBundle\Annotation as GraphQL;

/**
 * @GraphQL\Subscription()
 * @GraphQL\Argument(name="user", type="ID!")
 */
class OnUpdateUser
{
    /**
     * @var EntityManagerInterface
     */
    protected $em;

    /**
     * @param EntityManagerInterface $em
     */
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

The above example use the argument `user` to ensure clients to subscribe for changes
for specified user, this parameter can be optional if you want to allow subscribers to listen
for any user change.

Now in the publisher must call the subscription with all filters specified.

````php
$this->publisher->publish(OnUpdateUser::class, ['user'=> $user], ['userId'=> $user->getId()]);
````

In the above filter only a subscriber for given user receive the update, in the other hand
if the user argument is optional, all subscribers without filters receive the update too.

In the following example dispatch subscriptions with many filters:

````php
$this->publisher->publish(
            OnUpdateUser::class,
            [
                'user' => $user,
                'company'=> $company,
                'country'=> $country,
            ],
            ['userId' => $user->getId()]
        );
````
Subscribers can be subscribed to all filters or only a part of them, for example, 
a client can be observing all users of a certain country, while another one of a certain 
company and country. 

Filters are applied using `AND` operator and all filters must match in the client side.
For example, the user is observing for specific country and company, if only the
company match the subscription is not dispatched for that client.

Can define filters as arrays to allow clients to observe multiple items, example:

````
/**
 * @GraphQL\Subscription()
 * @GraphQL\Argument(name="user", type="[ID]!")
 */
 class OnUpdateUser
 {
````

In the above example if the subscriber is subscribed to multiple items the subscription
will be dispatched if any item match. For example, can listen for updates for multiple
users and receive a update if one of them is modified.

# Dynamic Filters

In some cases you may need apply filters dynamically based on current user permissions.
For example; a client can only listen for users updates under his company. 

In this cases instead of add a required argument by the client
you must add this filter dynamically in the subscription.

To accomplish this your subscription must implements `Ynlo\GraphQLBundle\Subscription\FilteredSubscriptionInterface`

````php
namespace App\Subscription\User;

use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Subscription\FilteredSubscriptionInterface;

/**
 * @GraphQL\Subscription()
 */
class OnUpdateUser implements FilteredSubscriptionInterface
{
    /
    /....
    /

    public function getFilters(): array
    {
        return ['company' => $this->currentUser->getCompany()];
    }
}
````
Now your subscription only will be dispatched if the company match

````
$this->publisher->publish(OnUpdateUser::class, ['company'=> $company ], ['userId' => $user->getId()]);
````

# Null Return

Return a null value in the subscription is not really a filter and should be used as a last resource, because
the subscription is already processed and increase the server load. Saying that, you can return a `null`
value in the subscription to avoid dispatch a update to some subscriber.


````php

namespace App\Subscription\User;

use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Ynlo\GraphQLBundle\Annotation as GraphQL;

/**
 * @GraphQL\Subscription()
 * @GraphQL\Argument(name="user", type="ID!")
 */
class OnUpdateUser
{
    //...

    public function __invoke($userId)
    {
        if (!$this->currentUser->isAdmin()){
            return null;
        }
 
        return $this->em->getRepository(User::class)->find($userId);
    }
}
````

