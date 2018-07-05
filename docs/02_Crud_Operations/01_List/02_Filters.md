GraphQLBundle provides a generic system to apply filters on collections of nodes.

> Advanced filters are available since `v1.2`. In previous versions filters are used
inside `filters` argument and does not have any configuration.

By default, if a field is exposed and is related to a entity property (non virtual field), 
a filter with the same name is automatically created and exposed in the list.

For example, given the following node:

````php
<?php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Symfony\Component\Validator\Constraints as Assert;
use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Model\NodeInterface;

/**
 * @ORM\Entity()
 * @ORM\Table()

 * @GraphQL\ObjectType()
 * @GraphQL\QueryList()
 */
class Post implements NodeInterface
{

    /**
     * @var int
     *
     * @ORM\Id
     * @ORM\Column(name="id", type="integer")
     * @ORM\GeneratedValue(strategy="IDENTITY")
     */
    protected $id;

    /**
     * @var User
     *
     * @Assert\NotNull()
     *
     * @ORM\ManyToOne(targetEntity="App\Entity\User", inversedBy="posts")
     * @ORM\JoinColumn(onDelete="CASCADE")
     */
    protected $author;

    /**
     * @var string
     *
     * @Assert\NotBlank()
     *
     * @ORM\Column(name="title", type="string")
     */
    protected $title;

    /**
     * @var string
     *
     * @Assert\NotBlank()
     *
     * @ORM\Column(name="body", type="string", nullable=true)
     */
    protected $body;
````

In this case 3 filters are automatically created `author`, `title` and `body`. 
API consumers can use these filters to reduce collection results.

<div class="graphiql">
<div class="request">

````graphql
query post($where: AllPostsCondition) {
  posts {
    all(first: 10, where: $where) {
      totalCount
      pageInfo {
        startCursor
        endCursor
        hasNextPage
        hasPreviousPage
      }
      edges {
        cursor
        node {
          id
          title
          body
        }
      }
    }
  }
}
````

<div class="variables">

```json
{
  "where": {
    "title": {
      "op": "CONTAINS",
      "value": "lorem"
    },
    "body":{
      "op": "CONTAINS",
      "value": "itsum"
    },
    "author":{
      "op": "IN",
      "nodes": ["VXNlcjox"]
    }
  }
}
````

</div>
</div>
<div class="response">

```json
{
  "data": {
    "posts": {
      "all": {
        "totalCount": 2,
        "pageInfo": {
          "startCursor": "Y3Vyc29yOjA=",
          "endCursor": "Y3Vyc29yOjE=",
          "hasNextPage": false,
          "hasPreviousPage": false
        },
        "edges": [
          {
            "cursor": "Y3Vyc29yOjA=",
            "node": {
              "id": "UG9zdDo4",
              "title": "Qui nostrum doloremque minima aut molestiae sapiente.",
              "body": "Tempore similique ut debitis consequatur. Dolorum doloremque quasi vero nobis error itsum fuga ut. Quia quasi sit dolore ad sunt est. Itaque cumque officiis ut quis quisquam consequatur asperiores. Magnam nostrum ea corrupti."
            }
          },
          {
            "cursor": "Y3Vyc29yOjE=",
            "node": {
              "id": "UG9zdDo5",
              "title": "Voluptas lorem esse dolor qui illo.",
              "body": "At itsum enim tempora voluptas. Dolorem voluptates deserunt provident natus ipsam. Est ipsam quia reprehenderit sint mollitia sed facere. Sit delectus ad iusto molestias iusto. Laboriosam nulla earum eius repellat culpa. Harum voluptatem sit nihil laboriosam sed. Nobis hic rerum delectus dolorum voluptas cupiditate aut consequatur. Ullam qui ea voluptatem aut cum vitae nostrum. Maiores non omnis aut quos ut ad est quidem. Rerum voluptates laboriosam ea porro blanditiis."
            }
          }
        ]
      }
    }
  }
}
````

</div>
</div>

At this point no special configuration is required and different filters are automatically 
created for each type of column.

As mentioned above, all filters are created automatically, but you can change this behavior.
For example; if you want only expose some filters.

Explicitly define available filters:

````
@GraphQL\QueryList(filters={"*": false, "title,body": true})
````

Explicitly ignore filters:

````
@GraphQL\QueryList(filters={"*", "author": false})
````

# Customizing filters


## Generic Type

The best filter type is automatically guessed for each field, in most cases this is enough, 
but if you want can create your own filter type. 

For example, the `title` property automatically use the `StringFilter` and this filter allow 
advanced conditions like `CONTAINS`, `START_WITH` etc.. to do advanced searches. If you want
to use another options, or make the filter for this field more simple can create your own filter type.

> A filter type must be a generic filter that can be used in many collections.

For example, we can create a a generic type called `LikeFilter` as simple filter
to apply to string fields using always `LIKE %%` in the query. This filter does not have other
options and only require a string as argument.

````php
<?php
namespace App\Filter;

use Doctrine\ORM\QueryBuilder;
use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Filter\FilterContext;
use Ynlo\GraphQLBundle\Filter\FilterInterface;

/**
 * @GraphQL\Filter(type="string")
 */
class LikeFilter implements FilterInterface
{
    public function __invoke(FilterContext $context, QueryBuilder $qb, $condition)
    {
        if (!$context->getField() || !$context->getField()->getName()) {
            throw new \RuntimeException('There are not valid field related to this filter.');
        }

        $alias = $qb->getRootAliases()[0];
        $column = $context->getField()->getOriginName();
        $qb->andWhere("{$alias}.{$column} NOT LIKE '%{$condition}%'");
    }
}
````

>> Filters must implements `Ynlo\GraphQLBundle\Filter\FilterInterface` and the `@Filter`
annotation is required with at least the filter type.

Now we can use this type to override the default filter for fields like `title` or `body`

````
@GraphQL\QueryList(filters={"*", "title,body": "App\Filter\LikeFilter") })
````

Now can use a simple query like this in our API:

````
{
  "where": {
    "title": "Lorem"
  }
}
````

In the above example the `LikeFilter` has `type="string"`, 
this means that the filter only allow a string as argument. 
Can use any `InputObject` or `scalar` as input type for filters.

For example; to accept a object as argument firstly create a input object:

````php
<?php

namespace App\Model\Filter;

use Ynlo\GraphQLBundle\Annotation as GraphQL;

/**
 * @GraphQL\InputObjectType()
 */
class LikeAdvancedExpression
{
    /**
     * @GraphQL\Field(type="string")
     *
     * @var string
     */
    protected $value;

    /**
     * @GraphQL\Field(type="boolean")
     *
     * @var boolean
     */
    protected $negativeLike = false;
````

Then update your filter:

````php
<?php
namespace App\Filter;

use Doctrine\ORM\QueryBuilder;
use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Filter\FilterContext;
use Ynlo\GraphQLBundle\Filter\FilterInterface;

/**
 * @GraphQL\Filter(type="string")
 */
class LikeFilter implements FilterInterface
{
    public function __invoke(FilterContext $context, QueryBuilder $qb, $condition)
    {
        if (!$context->getField() || !$context->getField()->getName()) {
            throw new \RuntimeException('There are not valid field related to this filter.');
        }

        $alias = $qb->getRootAliases()[0];
        $column = $context->getField()->getOriginName();
        if ($condition->isNegativeQuery()){
            $qb->andWhere("{$alias}.{$column} NOT LIKE '%{$condition->getValue()}%'");
        } else {
            $qb->andWhere("{$alias}.{$column} LIKE '%{$condition->getValue()}%'");
        }
    }
}
````

And your queries should now look like this:

````
{
  "where": {
    "title": {
        "value": "Lorem",
        "negativeLike": true
    }
  }
}
````

## Custom Filter

Alternatively to a generic filter type can create a custom filter only applicable to a specific node or interface. 

Using the same previous **Post** entity, imagine that you need a filter like `hasComments` to display 
only posts with or without comments. In this case you need create a new filter.

````php
<?php
namespace App\Filter\Post;

use Doctrine\ORM\QueryBuilder;
use Ynlo\GraphQLBundle\Annotation as GraphQL;
use Ynlo\GraphQLBundle\Filter\FilterContext;
use Ynlo\GraphQLBundle\Filter\FilterInterface;

/**
 * @GraphQL\Filter(
 *     type="bool",
 *     description="View only posts with or without comments"
 * )
 */
class HasComments implements FilterInterface
{
    public function __invoke(FilterContext $context, QueryBuilder $qb, $condition)
    {
        $alias = $qb->getRootAliases()[0];
        $qb->leftJoin("{$alias}.comments", 'comments');

        if ($condition) {
            $qb->andWhere("{$alias}.comments is not empty");
        } else {
            $qb->andWhere("{$alias}.comments is empty");
        }
    }
}
````

> Like other classes filters use [naming convention](../../08_Reference/02_Naming_Conventions.md) 
and a class inside `\Filter\Post\...` namespace are automatically applied to any list related to `Post` node.

Now a filter called `hasComments` is available in the list of posts.

Example to filter posts with comments:
````json
{
  "where": {
    "hasComments": true
  }
}
````
