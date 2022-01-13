---
title: "Don't fear local variables 😱"
date: 2022-01-13T18:52:32+01:00
draft: false
---

Local variables are one of the basic building blocks in programming in general.
In Ruby, it is the only variable that doesn’t start with special characters like @, @@, or $,
which indicates that they should be used often. However, many Rubyists, including me a couple of months ago,
tend to avoid them in favor of message chains, passing expressions directly to the caller, or by using the extract_method pattern

Here are some simple examples of avoiding local variables

#### Message Chains

The Example below is not that bad at the end of the day, however, it's very common and I believe we can do better
Do you see that `Rails.logger...`? That's what we are talking about

```ruby
def feed_animal(animal)
  case animal
  when :dog
    feed_the_dog
    Rails.logger.info("Dog is full now")
  when :cat
    feed_the_cat
    Rails.logger.info("Cat is full now")
  else
    Rails.logger.error("The breed does not exist")
  end
end

if Rails.env.development? || Rails.env.test?
```

#### Directly passing expressions

This scenario is quite bad, the code is much less readable.

```ruby
Success(DTO::Task.new(
  task.attributes.symbolize_keys.merge(
	  author: task.author.attributes.symbolize_keys
  )
))

ProjecMenagment::Issues::DTO::IssueField.new(
  filed: { description: "Descirption", title: "Title", user: {
	    usnermame: "user", email: "user@example.com"
		}
	},
  issue_field_definition: ProjecMenagment::Issues::DTO::IssueFieldDefinition.create(
		issue_field_definition_params
	)
)
```

## Why local variabels

As we've seen, avoiding local variables doesn't make code more readable, rather opposite... . Especially if we directly pass expression with long message chains to the caller.
The second reason, which probably won't be a problem unless we do something computation heavy... is when we access a local variable, the virtual machine knows the location of the local
variable and can more easily access memory.

## How to refactor
Well, that should be fairly simple, just [extract variable](https://refactoring.guru/extract-variable).

So in our first example we could just extract `Rails.logger` to variable or inject the logger to the class/method with the assigned default value

```ruby
def feed_animal(animal, logger = Rails.logger)
  case animal
  when :dog
    feed_the_dog
    logger.info("Dog is full now")
  when :cat
    feed_the_cat
    logger.info("Cat is full now")
  else
    logger.error("The breed does not exist")
  end
end
```
Second code snippet might look like this

```ruby
task = task.attributes.symbolize_keys
author = task.author.attributes.symbolize_keys
task_with_author = task.merge(author: author)
task_DTO = DTO::Task.new(task_with_author)

Success(task_DTO)
```

## Little Bonus
Here comes a more interesting example. The code we will take a look at is not just fine, it is excellent, but let's see if we can do even better.
So what gets our attention here... are the private methods(user, topic, video) called directly in `send_weekly_iteration_mailer`. 
I often have the feeling that `extract_method pattern` is pushing us away from using local variables, sometimes it's good sometimes not 

Let's take a look at the class below, the author decided to abstract calls to AR models to private methods instead of calling them directly inside
`send_weekly_iteration_mailer` which is obviously a good thing, but if the class will be a bit bigger and `send_weekly_iteration_mailer` will call other methods, that 
will compose of other methods it might get a bit confusing, but still a very clean piece of code though

```ruby
class WeeklyMailer

  def initilize(user_id, video_id, topic_id)
   	@user_id = user_id
   	@video_id = video_id
   	@topic_id = topic_id
  end

  attr_reader :user_id, :video_id, 

  def send_weekly_iteration_mailer
    WeeklyMailer
      .update(user: user, video: video, topic: topic)
      .deliver_now
  end

  # Other methods... 

  private

  def user
    User.find(user_id)
  end

  def video
    Video.find(video_id)
  end

  def topic
    Topic.find(topic_id)
	end
end
```

But maybe we can combine hiding the calls to AR modes while still using local variables.
Personally I find it even more explicit, and that would be my way to go

```ruby
class WeeklyMailer

  def initilize(user_id, video_id, topic_id)
   	@user_id = user_id
   	@video_id = video_id
   	@topic_id = topic_id
  end

  attr_reader :user_id, :video_id, 

  def send_weekly_iteration_mailer
    user = find_user
    topic = find_topic
    video = find_video

    WeeklyMailer
      .update(user: user, topic: topic, video: video)
      .deliver_now
  end

  # Other methods... 

  private

  def find_user
    User.find(user_id)
  end

  def find_video
    Video.find(video_id)
  end

  def find_topic
    Topic.find(topic_id)
  end
end
```

## Usefull Links
* [Polished Ruby Programmer, 3rd chapter](https://www.packtpub.com/authors/jeremy-evans)
* [The Local Variable Aversion Antipattern](https://www.soulcutter.com/articles/local-variable-aversion-antipattern.html)
