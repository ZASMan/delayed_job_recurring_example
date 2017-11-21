# delayed_job_recurring_example

At some point in your work as a Rails developer, you'll need some automated background tasks. In this tutorial, you'll learn how to create a background process to send automated emails using the [`delayed_job_recurring`](https://github.com/amitree/delayed_job_recurring) and [`delayed_job_active_record`](https://github.com/collectiveidea/delayed_job_active_record) gems, how to load them up with the [`after_party`](https://github.com/theSteveMitchell/after_party) gem, an example of how to create a model for the logging whether or not the jobs run in the database, and how to assure that the emails only send once.

Let's take an agile approach and approach this task as a user story:

*As a an administrator of the application, I would like users to receive a single reminder email to confirm their accounts if they are not confirmed a week after registration, to assure that users confirm their accounts in a timely manner.*


+ The first thing you'll need to do is create a scope for those unconfirmed users. This could look like the following:
```
class User < ActiveRecord::Base
  scope :all_unconfirmed, -> { where(is_confirmed: false, account_created_at: Date.today - 1.week ) }
end
```

Now that we have a scope for the users, let's move onto the task.

+ Let's start with the necessary gems:

```
# Gemfile
# For Loading the Initial Delayed Job
gem 'after_party'

# For Executing the background processes
gem "daemons" # Dependency for delayed job
gem 'delayed_job_active_record'
gem 'delayed_job_recurring'
```

+ Before we move on to the recurring job, let's take a look at the example email that we'd like to run.  Here's an example of what it could look like:

```
class UserMailer < ApplicationMailer
  def registration_confirmation_reminder_email(user)
    @user = user
    email_address = @user.email
    mail(
      to: email_address,
      subject: "Reminder to Confirm Registration",
      template_path: 'user_mailer',
      template_name: 'registration_confirmation_reminder'
    )
  end
end
```

+ Now let's use the `delayed_job_recurring` gem to schedule this task to run every day. Here's an example, which is described more or less as in the documentation:

```
module Recurring
  class ScheduledReminderEmail
    include Delayed::RecurringJob
    run_every 1.day
    run_at '12:00am'
    timezone 'Eastern Time (US & Canada)'
    queue 'slow-jobs'
    def perform
      users = User.all_unconfirmed
      users.each do |user|
        UserMailer.registration_confirmation_reminder_email(user).deliver_now
      end
    end
  end
```

+ Now, lets use the after party gem to create a rake task to execute the recurring task. I like using after party because you can easily set it up to run `rake after_party:run` after every deployment.

```
namespace :after_party do
  desc 'Deployment task: send_registration_confirmation_emails'
  task registration_confirmation_emails: :environment do
    puts "Running deploy task 'registration_confirmation_emails'"
    Recurring::ScheduledReminderEmail.schedule!
  end
end

```

+ This should set up the task to run after every deployment, and solves most of the criteria. But we also have to answer another part of the user story, the `user` part- the experience of the administrator: that the user should receive a **single** email. In addition, how can we assure to the administrator that the emails are being sent out to recently created users? Moreover, with all of this happening on the backend on the server, how can we as a developer assure that the background process assure that the recurring job isn't being created multiple times after deployment?
