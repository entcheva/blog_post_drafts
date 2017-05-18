# ActiveRecord::Observers 101

> *"A common side effect of partitioning a system into a collection of cooperating classes is the need to maintain consistency between related objects. You don't want to achieve consistency by making the classes tightly coupled, because that reduces their reusability."*  
>
> Design Patterns: Elements of Reusable Object-Oriented Software

Active Record Observers are a great tool for when we want to monitor an event occurence and take action, and especially when that event concerns multiple classes.

### Active Record callbacks vs. ActiveRecord::Observers
Active Record's built in callback [helper methods](http://guides.rubyonrails.org/active_record_callbacks.html#available-callbacks) can easily hook into the object lifecycle to affect change. However, when callbacks affect objects outside the model, we end up violating the single responsibility principle, so we try to avoid them if we can. Using observers brings back single responsibility to models while still allowing us to still hook into the object lifecycle.

The Observer is a central place outside of the original class to watch for database changes. By extracting that logic outside of the model, we bring back single responsibility and avoid commingling models. Observers only report or take action on a process, and should not affect a process directly.

[ActiveRecord::Observer](http://api.rubyonrails.org/v3.2/classes/ActiveRecord/Observer.html) is an implementation of the [Observer pattern](https://en.wikipedia.org/wiki/Observer_pattern). It was removed from core in Rails 4.0 but is now [available as a gem](https://github.com/rails/rails-observers).

##### Example: Callbacks

In the example below, we're referencing `MessageMailer` in the `User` class, increasing the `User` class's responsibility:

```
class User
  after_create :notify

  protected
    def notify
      MessageMailer.send_welcome_email(self).deliver
    end
end
```

##### Example: Observers

Alternatively, by using observers in the example below, we extract the `MessageMailer` logic out of `User` and put it in its own class, `UserObserver`, that watches for an event and dispatches an action accordingly:

```
class UserObserver < ActiveRecord::Observer
  def after_create(user)
    MessageMailer.send_welcome_email(user)
  end
end
```

##### Example: Observing multiple classes

Observers are especially helpful in cases like the below where there are multiple models commingled and we want to hook into all of them to watch for some event. We can pull logic out of models where it doesn't belong, put it in its own separate place, observe actions, and still affect change as needed.

```
class AuditObserver < ActiveRecord::Observer
	observe :account, :balance

	def after_update(record)
		AuditTrail.new(record, "UPDATED")
	end
end
```

### AR::Observers and invisibility

One thing to keep in mind with observers is that when we use them, since there is no visible reference to the observer in the observed class, we now have events that are not readily apparent in our observed class. 

Due to these unseen side effects, it can be less clear what's happening. If there's a bug, it can be hard to track, since there is no direct link to the thing that is observing it.

### When to use AR::Observers

Active Record Observers are useful when we:
- want to stay updated on a certain process  
- want to maintain consistency  
- have multiple classes involved in an event  

They require very little [setup](https://github.com/rails/rails-observers#installation), and since they are Active Record objects, they come rolled up with the functionality of Active Record. 

Good use cases for Observers:
- cached values  
- invalidate caches  
- tk  
- ...  

### When *not* to use AR::Observers

It's also possible to overuse observers. 

If there is just one thing that happens, we probably don't need to use observers. For example, our first example of sending a welcome email after a new user is created is probably simple enough that it doesn't actually need an observer. The same use case can be handled by extracting a service object.

However, if you want to take additional actions after the user is create (for example, anything from the list above), it may be worth using an observer. 

...

### References

[Rails::Observers](https://github.com/rails/rails-observers)  
[Giant Robots Smashing Into Other Robots: 13: I'll disagree in just a little bit](http://giantrobots.fm/13)  
[Upcase: The Weekly Iteration: Callbacks](https://thoughtbot.com/upcase/videos/callbacks)  
[Design Patterns In Ruby](https://www.amazon.com/Design-Patterns-Ruby-Russ-Olsen/dp/0321490452)  
[Design Patterns: Elements of Reusable Object-Oriented Software](http://a.co/7iVlVgw)
