One of the greater benefits of use subscriptions is to manage highly costly tasks, 
for example a webapp tells the server to compute a report and the server delegates the computation of the report to an 
asynchronous worker (using message queue), and closes the connection with the webapp the worker sends the report to 
the webapp when it is computed.

You have two ways to accomplish this task:

 - Create two operations, a mutation to request the report and subscription operation to listen when it's computed.
 - Or use the `AsynchronousJobInterface` to make this two actions with a one single operation.

````
use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Subscription\AsynchronousJobInterface;

/**
 * @GraphQL\Subscription()
 */
class UpdateReport implements AsynchronousJobInterface
{
    //...

    public function onSubscribe(): void
    {
        // use message queue to start the report build 
    }

    public function __invoke($reportId)
    {
       // return the report
    }
}
````
The method `onSubscribe` is executed synchronously on subscription and
the method __invoke is called asynchronously once the report completed using the following
code in anywhere in your application.


````
$this->publisher->publish(UpdateReport::class, [], ['reportId' => $reportId]);
````

