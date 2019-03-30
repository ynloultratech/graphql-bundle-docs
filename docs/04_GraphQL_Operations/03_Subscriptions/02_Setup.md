GraphQLBundle subscriptions use the [Mercure](https://mercure.rocks/) protocol for delivery & [Redis](https://redis.io)
server to store & manage subscriptions must install this dependencies to setup your project.

# Mercure Hub

> Mercure is a protocol allowing to push data updates to web browsers and other HTTP clients in a convenient, 
fast, reliable and battery-efficient way. It is especially useful to publish real-time
updates of resources served through web APIs, to reactive web and mobile apps.

First, [download and run a Mercure hub](https://github.com/dunglas/mercure#hub-implementation). Then, install the Symfony bundle:

    $ composer require mercure

Finally, 2 environment variables [must be set](https://symfony.com/doc/current/configuration/external_parameters.html):

- **MERCURE_PUBLISH_URL:** the URL that must be used by to publish updates & subscribe, Example: http://api.example.com:3000/hub
- **MERCURE_JWT_SECRET:** a valid Mercure JSON Web Token (JWT) allowing publish updates to the hub

The JWT must contain a empty `mercure.publish` property. 
Example [publisher JWT](https://jwt.io/#debugger-io?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXJjdXJlIjp7InN1YnNjcmliZSI6W10sInB1Ymxpc2giOltdfX0.0NJJunphS4jWTqU7uvIEakLo3jnLuAb4SqD8PCSYxW4)
 (demo key: `!UnsecureChangeMe!`).

>> The Mercure sever must be configured to allow **ANONYMOUS** subscribers (`ALLOW_ANONYMOUS=1`)

# Redis

>  Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker.
  It supports various data structures such as Strings, Hashes, Lists, Sets etc.
  
Redis is used to store each subscription request per client (query, authentication, endpoints etc.), 
in order to execute this request async later when a subscription update is dispatched.

Install a [redis server](https://redis.io/download#installation) in your system and the redis php extension.

By default subscriptions are configured to use a local redis server but 
you can change these settings:

````yaml
graphql:
    # Manage subscriptions settings
    subscriptions:
        # Configure redis server to use as subscription handler
        redis:
            host:                 localhost
            port:                 6379

            # Define custom prefix to avoid collisions between applications
            prefix:               'GraphQLSubscription:'

````

# Subscription consumer

The following command must be running in order to consume subscriptions to send to Mercure hub.

    php bin/console graphql:subscriptions:consume