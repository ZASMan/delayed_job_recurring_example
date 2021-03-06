# delayed_job_recurring_example

At some point in your work as a Rails developer, your application will go beyond a trivial MVC architecture to require some automated processes to run in the background. In this tutorial, we'll apply agile development principles and user-centric design to fulfill the needs of the task by approaching it as a user story. We will:

1. Create a background process to send automated emails using the [`delayed_job_recurring`](https://github.com/amitree/delayed_job_recurring) and [`delayed_job_active_record`](https://github.com/collectiveidea/delayed_job_active_record) gems
2. Set up a background process to load them up with the [`after_party`](https://github.com/theSteveMitchell/after_party) gem and assure that the delayed jobs to run the background processes are only created once.
3. Design an `EmailLog` model to assure that the emails are only sent once.
4. Design a `JobReport` model to help keep track of the delayed jobs in the database.
5. Plan ahead, and take into consideration design, especially when changing your database.

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

+ First, create an `EmailLog` model to log all emails being sent. **Note:** Because it seems logical that other emails might be sent for other objects in the future (I.E. the email log can belong to more than just a 'user', lets utilize polymorphic association to create the email log object, and have it belong to an `email_loggable` object. Here's what the migration could look like:

```
# The 'description' will help when you create a view for this and
# provide an accurate description of the email task to your end user, the administrator
class CreateEmailLogs < ActiveRecord::Migration[5.0]
  def change
    create_table :email_logs do |t|
      t.string :email_type
      t.string :email_loggable_type
      t.text :description
      t.references ::email_loggable_id, foreign_key: true

      t.timestamps
    end
  end
end
```
+ The relationship between the user/email log can look like so:
```
class EmailLog < ActiveRecord::Base
  belongs_to :email_loggable, polymorphic: true
  belongs_to :user, foreign_key: 'email_loggable_id', class_name: 'User'

  EMAIL_LOG_TYPES = {
    confirmation_reminder: 'user confirmation email'
  }.freeze

  # Here in the future you might have email loggable types as 'post' or 'comment' or whatever you'd like emails for
  EMAIL_LOGGABLE_TYPES = {
    user: 'user',
  }.freeze
end

class User < ActiveRecord::Base
  has_many :email_logs, as: :email_loggable, dependent: :destroy
  scope :all_unconfirmed, -> { where(is_confirmed: false, account_created_at: Date.today - 1.week ) }
end


```

+ We should then create an `EmailLog` record each time after an email is sent. Because this could be used in multiple mailers, let's add it to the `ApplicationMailer` so all other mailers can inherit it.


```
class ApplicationMailer < ActionMailer::Base
  def create_email_log(email_type, email_address, loggable_id, loggable_type)
    EmailLog.create!(
      email_type: email_type,
      email_loggable_id: loggable_id,
      email_loggable_type: loggable_type,
      description: "sent to #{email_address} on #{Date.today}."
    )
  end
end
```

+ Now let's add that method to be executed at the end of the user mailer method to create the record.

```
  def registration_confirmation_reminder_email(user)
    @user = user
    email_address = @user.email
    email_type = 'user confirmation email'
    loggable_id = @user.id
    loggable_type = 'user'
    mail(
      to: email_address,
      subject: "Reminder to Confirm Registration",
      template_path: 'user_mailer',
      template_name: 'registration_confirmation_reminder'
    )
    create_email_log(email_type, email_address, loggable_id, loggable_type)
  end
```

+ Now assuring that the email sends only once becomes possible. Let's add the following to our `apps/models/user.rb`:
```
  def email_already_sent?(email_type)
    EmailLog.where(email_loggable_id: id, email_type: email_type).present?
  end
  ```
+ And update the mailer to stop the mailer method if the email has indeed been sent:

```
  def registration_confirmation_reminder_email(user)
    @user = user
    email_address = @user.email
    email_type = 'user confirmation email'
    loggable_id = @user.id
    loggable_type = 'user'
    # This will stop the email from being sent
    return false if user.email_already_sent?(email_type)
    mail(
      to: email_address,
      subject: "Reminder to Confirm Registration",
      template_path: 'user_mailer',
      template_name: 'registration_confirmation_reminder'
    )
    create_email_log(email_type, email_address, loggable_id, loggable_type)
  end
```

+ After that, you can create an index page for the email logs to display them to your end user.


+ Now, lets go back to the recurring jobs. If you recall, we want to add something to the front end to assure that the job is created. We can go about this by creating a new model, `JobReport`, and creating a job report each time the recurring task is successfully created. Design it however you want, ideally similar to the `EmailLog` object- like providing a description to the end user. 


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
  
    def create_report
      # Stop the creation of the report if the job already exists
      return false unless Delayed::Job.count.positive?
      # These will take info from the last job created.
      # Check the delayed job documentation/your database
      # For information on the Delayed::Job objects.
      last_job = Delayed::Job.last
      job_run_date = last_job.run_at.to_s
      job_run_time = last_job.run_at.hour.to_s + ":00"
      JobReport.create!(
        job_name: 'delayed user registrations reminder',
        description: "Recurring task created for delayed user registrations" \
        " created on deployment on #{Date.today}. Task will first run at #{job_run_time} on #{job_run_date}."
      )
    end
  end
```

+ Create a job report index page to display the report information to the end user.

+ Now, let's add that job report creation method to the after party task and also update the task to assure that the delayed job task won't be created multiple times in the database (since after party is run after each deployment):

```
namespace :after_party do
  desc 'Deployment task: send_registration_confirmation_emails'
  task registration_confirmation_emails: :environment do
    puts "Running deploy task 'registration_confirmation_emails'"
    # Inspect the Delayed::Job object in your database for more info on the handler atribute
    # This will delete the old version of the job and create a new one on each deployment
    job_handler_like = '(handler LIKE ?)', '--- !ruby/object:Recurring::ScheduledReminderEmail%'
    Delayed::Job.where(job_handler_like).destroy_all
    Recurring::ScheduledReminderEmail.schedule!
    # Create Job Report
    Recurring::ScheduledReminderEmail.create_report
    # Add an extra line for the command line for quick reference if you like
    puts 'Delayed Job Created for Scheduled Reminder Emails' if Delayed::Job.where(job_handler_like).present?
  end
end

```

+ And that's it! You'll have satisfied all criteria of the user story. Hope this provided some help on how to use the delayed jobs gem and design it with a user centered approach!
