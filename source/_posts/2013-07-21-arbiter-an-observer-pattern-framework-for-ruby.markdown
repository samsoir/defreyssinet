---
layout: post
title: "Arbiter: A simple notification observer framework for Ruby"
date: 2013-07-21 11:59
comments: false 
categories: [ruby, arbiter]
---

The team at [Sittercity](http://www.sittercity.com) have been busy building a vast variety of software components for the organisation over the last twelve months. The array of components crafted range from iOS applications to single sign-on services. One problem that has consistently appeared is how to handle various use cases where multiple parties are interested in the result of a particular routine. Early in the lifecycle of a project these concerns are usually handled directly within the use case itself, and providing there are only one or two operations things are usually happy. But as it turns out, many use cases have a wide range of interested parties. Shovelling each interested parties concerns into the use case does not scale well, whilst simultaneously violating a number of good design practices.

To help explain the problem, consider this simple example.

<!-- more -->

``` ruby lib/authenticate/context/email.rb
module Authenticate
  module Context
    class Email

      def initialize(user_repository, secret_hashing_policy)
        @user_repository = user_repository
        @secret_hashing_policy = secret_hashing_policy
      end

      def execute(email, secret)
        authentic = false
        user = @user_repository.find_by_email(email)

        if user
          authentic = user.secret.eql?(@secret_hashing_policy.hash(secret))
        end

        return authentic
      end
    end
  end
end
```

Here we have a simple context that authenticates credentials based on the supplied email and secret. The context has two dependencies, as shown in Fig.1;

{% img /images/arbiter-dependency-graph-1.png Fig.1 Dependency graph for Authenticate::Context::Email %}

1. A user repository that provides access user data
2. A hashing policy that governs how secrets are hashed before storage

Both of these dependencies are passed into the context during construction. The context provides a single `execute` method that tests the users authenticity based on the user identifier and secret provided.

This use case as it stands is doing its job very well. Additionally it is adhering to the [single responsibility principle](http://en.wikipedia.org/wiki/Single_responsibility_principle), as it only concerns itself with establishing the authenticity of the credentials supplied. It isn't concerned with loading user data or how to correctly hash a secret. 

Later on, the business owner walks over and asks to have insight into the number of authentications occurring;

{% blockquote %}
When a user authenticates, the business must record the authentication data into our statistics data warehouse. We want to know who was authenticating and the result of their attempt. 
{% endblockquote %}

She wants to capture this data in another statistics repository owned by the Data Warehouse team. Given the data our business owner wants is related to authentications, it seems a simple requirement to fulfil.

``` ruby lib/authenticate/context/email.rb
module Authenticate
  module Context
    class Email
	
      def initialize(user_repository, secret_hashing_policy, stats_repository)
        @user_repository = user_repository
        @secret_hashing_policy = secret_hashing_policy
        @stats_repository = stats_repository
      end

      def execute(email, secret)
        authentic = false
        user = @user_repository.find_by_email(email)

        if user
          authentic = user.secret.eql?(@secret_hashing_policy.hash(secret))
        end

        @stats_repository.register_entry(email, authentic)

        return authentic
      end
    end
  end
end
```

Fulfilling the requirement to provide authentication statistics to the business has grown the authentication context slightly. As it now authenticates and updates statistics, you could argue that it has begun to violate the single responsibility principle. I would certainly agree with your argument if you were to make it. But given the scope of this change, most developers could probably live with that slight violation at this point.

A few days later, the business owner returns to ask for another great idea. 

{% blockquote %}
When a user fails to authenticate, they should receive an email notification instructing them on how to access the service. 
{% endblockquote %}

Now we could just go back and modify the use case to include a mailer and the context would grow further. We are definitely going to violate the single responsibility principle now, as we are authenticating credentials and performing other notification tasks. A another approach would be to create an additional use case for authenticating credentials and notifying various parties. This use case would be a composite of the `Authenticate::Context::Email` context and two others. It would certainly be cleaner than trying to overload the existing use case with all these extra responsibilities. However, either solution starts introducing dependencies that authentication should not have.

{% img /images/arbiter-dependency-graph-2.png Fig.2 Dependency graph for Authenticate::Context::Email demonstrating violation of the single responsibility principle %}

Fig.2 demonstrates that authenticating a set of credentials now requires a lot of knowledge about other unrelated parts of the system. The simple act of authentication has become confused by the other requirements that have been made.

At this point we should pause for a second and consider the problem again. What we really want to do is authenticate some credentials and then tell the system we are done _authenticating_. The system can then respond to the message from the authenticate context without having to provide knowledge of the intended behaviour.

We would like is the authentication context to execute and return a boolean value as it does now. Additionally, we want it to notify observers that is finished. The context would still require one additional dependency for notifying, but that would be it. The business owner could provide additional requirements in future for what should happen when authentication succeeds or fails, and the existing authentication context would not have to be altered.

Lets go back to our original implementation of `Authenticate::Context::Email` and add a basic concept of a notifier.

``` ruby lib/authenticate/context/email.rb
module Authenticate
  module Context
    class Email

      AUTHENTICATION_NOTIFICATION = 'authenticate::context::email::complete'

      def initialize(user_repository, secret_hashing_policy, notifier)
        @user_repository = user_repository
        @secret_hashing_policy = secret_hashing_policy
        @notifier = notifier
      end

      def execute(email, secret)
        authentic = false
        user = @user_repository.find_by_email(email)

        if user
          authentic = user.secret.eql?(@secret_hashing_policy.hash(secret))
        end

        @notifier.publish(AUTHENTICATION_NOTIFICATION, {
          'authentic' => authentic,
          'email' => email
        })

        return credentials_are_authentic
      end
    end
  end
end
```

Now we have introduced one additional dependency to the original two we defined. The notifier must be passed in, along with the user repository and hashing policy. But the notifier is the only additional dependency we will need. Now we can go and change the implementation of our statistics reporter and account access notifier to listen to the `authenticate::context::email::complete` notification coming from the notifier and perform their actions accordingly.

{% img /images/arbiter-dependency-graph-3.png Fig.3 Dependency graph for Authenticate::Context::Email demonstrating dependency inversion %}

This implementation also does something interesting to the dependencies. Notice that `Authenticate::Context::Email` depends on the `Notifier`, but no longer depends on `Authenticate::Stats` or `Message::AccountAccess`. Instead, `Authenticate::Stats` and `Message::AccountAccess` depend on `Notifier`. The result is that `Authenticate::Context::Email` no longer has any knowledge of `Authenticate::Stats` or `Message::AccountAccess` and vice versa. We have inverted the dependencies and introduced a solid boundary between our authentication use case and other parties that need to know about authentication notifications but are not part of that domain.

The `Notifier` part of this implementation is currently an abstract concept. After looking around the Ruby world for an existing solution, we ended up devising and building a new observer pattern framework called [Arbiter](http://github.com/sittercity/arbiter/).

## Introducing Arbiter

Arbiter is a simple notification framework. It provides a lightweight notification protocol that can be used across any Ruby application. Simply put, Arbiter is our solution to the `Notifier` class in the earlier example. There are only two component parts to Arbiter, the Arbiter and the Eventer. The Arbiter observes notifications and executes business logic associated with those notices.

The Eventer is the interface to post notices for observation. The Eventer consists of a transport (`Eventer.bus`) provided by the Arbiter and the `Eventer.post()` class method.

The Arbiter provides the transport for notifications posted by the Eventer. The Arbiter intercepts notifications on the transport and passes them to observers that register to a particular event. Registration of observers is done using the `Arbiter.add_listener()` class method. 

Any class can observe notifications from Arbiter, provided it conforms to the Arbiter protocol by implementing the following methods;

- `notify(message_name, args)`, called by Arbiter whenever it receives a notification that the observer is interested in.
- `subscribe_to()`,  used by Arbiter to establish the notifications that want to be observed. This method should return an array of notification names.

This is all that is required to begin observing notifications with Arbiter. Did I mention it was simple?

## Example

Revisiting the earlier example of an authentication context, now we will implement the context and the observers using the Arbiter framework.

First of all we need to decide on a transport for Arbiter to use. Given we want to keep this solution simple for the time being, lets use the standard Arbiter transport which uses an in-process execution model. To configure this, we'll setup the `Eventer.bus` in a configuration for the application.

``` ruby /config/eventer.rb
require 'eventer'
require 'arbiter'

Eventer.bus = Arbiter
```

The configuration file sets up Eventer to use the basic Arbiter as the transport, referred to as the `bus`. With this we can write the observers.

``` ruby /lib/stats/observer.rb
require 'stats/repository'

module Stats
  class Observer
    
    def initialize(stats_repository)
      @stats_repository = stats_repository
    end

    def subscribe_to()
      return [:authenticate]
    end

    def notify(message_name, args)
      @stats_repository.register_entry(args['email'], args['authentic'])
    end

  end
end
```

This observer provides the same functionality for recording authentications as implemented earlier. The actual recording of statistics is now completed within the observers `notify()` method.

We can use the same pattern for the observer that is responsible for the account access email message when authentication fails.

``` ruby /lib/message/authentication/observer.rb
module Message
  module Authentication
    class Observer
    
      def initialize(message_manager)
        @message_manager = message_manager
      end

      def subscribe_to()
        return [:authenticate]
      end

      def notify(message_name, args)
        @message_manager.send_reaccess_account(args['email']) if (args['authentic'] == false)
    end

  end
end
```

Now we have our two observers configured and ready to receive messages, lets update the original `Authenticate::Context::Email` context to use Arbiter.

``` ruby lib/authenticate/context/email.rb

module Authenticate
  module Context
    class Email

      def initialize(user_repository, secret_hashing_policy, eventer)
        @user_repository = user_repository
        @secret_hashing_policy = secret_hashing_policy
        @eventer = eventer
      end

      def execute(email, secret)
        authentic = false
        user = @user_repository.find_by_email(email)

        if user
          authentic = user.secret.eql?(@secret_hashing_policy.hash(secret))
        end

        @eventer.post(:authenticate, {
          'authentic' => authentic,
          'email' => email
        })

        return credentials_are_authentic
      end
    end
  end
end
```

Our context is now using the Arbiter `Eventer` to post the `:authenticate` notification to the transport. We are almost ready to go, but there is a problem. Right now we are publishing our notifications correctly and have two observers ready to observe. But they are not going to observe anything yet as they have not been registered to the Arbiter. To do this we will need to tell the Arbiter that `Stats::Observer` and `Message::Authentication::Observer` want to listen to notifications. Where you do this in your application will depend on its structure. But for the sake of simplicity lets set up these observers in the `eventer.rb` configuration file we defined earlier.

``` ruby /config/eventer.rb
require 'eventer'
require 'arbiter'
require 'lib/stats/observer'
require 'lib/message/authentication/observer'

Eventer.bus = Arbiter

Arbiter.set_listeners([Stats::Observer, Message::Authentication::Observer])
```

We have successfully implemented the Arbiter framework to replace our Notifier abstraction used earlier. As a result the authentication use case is now maintaining its single responsibility of authenticating credentials and the two observers we wrote handle the recording of statistics and any other messages as a result of the authentication process.

## Notifications beyond the process boundary

The example above uses Arbiter to send notifications in-process to observers. However there are times where there are concurrent processes operating with observers interested in a particular topic. 

For example, a main application handles user requests, while a second process is devoted to statistical analysis and a third is devoted to messaging users. The primary process will be publishing notifications and the other two processes want to listen to those notifications. In these cases the Arbiter example shown above will not work.

Having also encountered this problem, we have created two additional Arbiter implementations to allow for concurrent process observations.

1. `ResqueArbiter` uses the [Resque](http://github.com/resque/resque) framework as the `Eventer.bus`
2. `ZeromqArbiter` uses the [ZeroMQ](http://www.zeromq.org) (ØMQ) messaging protocol as the `Eventer.bus`. This Arbiter transport provides the most flexibility, but can be the most complex to use.

It is possible to use any transport for notifications, as long as you can provide the Arbiter implementation to handle the registering of observers and delivery of messages. Therefore users who want to use [RabbitMQ](http://www.rabbitmq.com) for example, should have no problem getting Arbiter to work with that messaging system.

However be aware that thread safety and other concurrent concerns still apply when using Arbiter across processes.

## Obtaining and using Arbiter

We welcome and encourage everyone to use and improve this lightweight notification framework. Please fork and pull to your (our our) delight.

Arbiter is now released as an open source library available on [GitHub](http://github.com/sittercity/arbiter) and pre-packaged as a Ruby [Gem](https://rubygems.org/gems/arbiter). Arbiter is licensed under the [ISC License](http://opensource.org/licenses/ISC).

Arbiter is maintained by the awesome Sittercity development team and was written by Robert Grider ([@mrerrormessage](http://twitter.com/mrerrormessage)) and Jeremy Bush ([@zombor](http://twitter.com/jeremybush)), based on a concept by Robert Grider and myself.
