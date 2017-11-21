# delayed_job_recurring_example
A simple example implementation of the delayed job recurring gem, executed with an after party task.


At some point in your work as a Rails developer, you'll need some automated background tasks. In this tutorial, you'll learn how to create a background process to send automated emails, how to load them up with the after party gem, and an example solution of how to keep track of whether or not these background jobs run. 

+ Let's start with the necessary gems:

```
# Gemfile
# For Loading the Initial Delayed Job
gem 'after_party'

# For Executing the background processes
gem "daemons"
gem 'delayed_job_active_record'
gem 'delayed_job_recurring'
```
