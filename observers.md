# ActiveRecord::Observers 101

> *"A common side effect of partitioning a system into a collection of cooperating classes is the need to maintain consistency between related objects. You don't want to achieve consistency by making the classes tightly coupled, because that reduces their reusability."*  
> 
> Design Patterns: Elements of Reusable Object-Oriented Software

The Observer pattern and Active Record Observers are a great tool for when you want to monitor an event occurence and take action, and especially when that event concerns multiple classes.

### Active Record callbacks vs. ActiveRecord::Observers
Active Record's built in callback [helper methods](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks) can easily hook into the object lifecycle to affect change. However, when callbacks concern multiple models, we end up violating the single responsibility principle, so we try to avoid them if we can. Using observers brings back single responsibility to models while still allowing us to still hook into the object lifecycle.  

The Observer is a central place outside of the original class to watch for database changes. By extracting that logic outside of the model, we bring back single responsiblity and avoid comingling models. Observers only report or take action on a process, and do not affect a process directly.

An observer is an implementation of the [Observer pattern](https://en.wikipedia.org/wiki/Observer_pattern). You can [write your own](https://github.com/nslocum/design-patterns-in-ruby/tree/master/observer), or you can use Rails' [ActiveRecord::Observer](http://api.rubyonrails.org/v3.2/classes/ActiveRecord/Observer.html), which was removed from core in Rails 4.0 but is now [available as a gem](https://github.com/rails/rails-observers).

In the example below, we're using a callback to initiate an action after creating a new user. We're mixing `MessageMailer` into the `User` class:

```
class User
  after_create :notify

  protected
    def notify
      MessageMailer.send_welcome_email(self).deliver
    end
end
```

As a better alternative, by using observers in the example below, we extract the `MessageMailer` logic out of `User` and put it in its own class, `UserObserver`, that watches for an event and dispatches an action accordingly:

```
class UserObserver < ActiveRecord::Observer
  def after_create(user)
    MessageMailer.send_welcome_email(user)
  end
end
```

Observers can also watch multiple classes:

```
class AuditObserver < ActiveRecord::Observer
	observe :account, :balance

	def after_update(record)
		AuditTrail.new(record, "UPDATED")
	end
end
```

Obervers are especially helpful in cases like the above where there are multiple models comingled and we want to hook into all of them to watch for some event. We can pull logic out of models where it doesn't belong, put it in its own separate place, observe actions, and still affect change as needed. 

### Accounting for unseen side effects

One thing to consider with observers is that we now have events that are not readily apparent in our observed class. Due to these unseen side effects, it can be less clear what's happening, and if there's a bug, it can be hard to track, since there is no direct link to the thing that is observing it. 

To mitigate this potential downside, we can write observers that reference the model they are observing. One way to do that is with `logger`. In this example from the [Rails documentation](http://api.rubyonrails.org/v2.3/classes/ActiveRecord/Observer.html), the observer uses `ActiveSupport::Logger` to log when specific callbacks are triggered:

```
class ContactObserver < ActiveRecord::Observer
    def after_create(contact)
      contact.logger.info('New contact added!')
    end

    def after_destroy(contact)
      contact.logger.warn("Contact with an id of #{contact.id} was destroyed!")
    end
  end
```

### Staying in sync

Another thing to consider is that since we're dealing with application state, it's possible that as the application transitions from one state to another, our observers can become out of sync.
 
For example, payment processing transactions usually have many steps. Consider a case where a transaction is created but the payment is then rejected. If we have an observer with an `after_create` callback, that callback is going to fire after `create`, regardless of whether that transaction is eventually rolled back. 

To account for this, we can use `before` and `after` hooks in callbacks to assure that we only inform observers after all changes have completed and the observer is heady to take action. 

View the [Active Record Callbacks documentation](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks) for a full list of callbacks in the order they will be executed.

### References

[Rails::Observers](https://github.com/rails/rails-observers)  
[Giant Robots Smashing Into Other Robots: 13: I'll disagree in just a little bit](http://giantrobots.fm/13)  
[Upcase: The Weekly Iteration: Callbacks](https://thoughtbot.com/upcase/videos/callbacks)  
[Design Patterns In Ruby](https://www.amazon.com/Design-Patterns-Ruby-Russ-Olsen/dp/0321490452)  
[Design Patterns: Elements of Reusable Object-Oriented Software](http://a.co/7iVlVgw)
