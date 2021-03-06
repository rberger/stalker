Stalker - a job queueing DSL for Beanstalk
==========================================

New in this [Fork](https://github.com/rberger/stalker)
-------------------------------------------------------

### Enable `Stalker.job` to access `Beanstalker::Job` instance

The `Stalker.job` can now have access to the `Beanstalk::Job` instance that was used to launch the `Stalker.job`. 

Added two more optional arguments to the end of the `Stalker.enqueue` argument list. 

If you use the existing signature for enqueue, there is no change in behavior and the Stalker.job still does not have access to the Beanstalk::Job instance.

    Stalker.enqueue(stalker_job, args={}, opts={})

The new calling signature to enable accessing the `Beanstalk::Job` instance in your `Stalker.job` is:

    Stalker.enqueue(stalker_job, args={}, opts={}, beanstalk_style=false, beanstalk_style_opts={})

If the last two args are not used or `beanstalk_style = false`, then the behavior of Stalker is the same as before.
If `beanstalk_style = true`, then when the `Stalker.job` is run by `Stalker.work`, the `Stalker.job` will get the instance of the `Beanstalk::Job` as the second argument after args.


The new signature for the `Stalker.job` declaration that can accept its `Beanstalk::Job` is:

    Stalker.job 'myjob' do |args, job, beanstalk_style_opts|
      # Body of Job
    end

The `job` contains the instance of `Beanstalk::Job` that was used to launch the running instance of the `Stalker.job` when it is run by `Stalker.work`. The `beanstalk_style_opts` is optional and contains the hash that was passed into `Stalker.enqueue`.

So your `Stalker.job` can do all the normal methods of `Beanstalk::Job` on the job that is passed in:

    require 'stalker' # Stalker 
    Stalker.job 'myjob' |args, job, opts| do
      log "args: #{args.inspect} opts: #{opts.inspect}"
      job.stats
      job.touch
      job.delete if opts['explicit_delete]
    end

#### Optional Over-rides ####

If you pass in true for the 4th argument of `Stalker.enqueue` and do not set `beanstalk_style_opts`, then the behavior of the Stalker.job is exactly the same as the old style, except your Stalker.job will have access to the Beanstalk:Job instance as its 2nd argument (after args).

You can use the 5th argument to `Stalker.enqueue` to pass in `beanstalk_style_opts` hash to override some of the Stalker behavior:

* `run_job_outside_of_stalker_timeout => true`, The job will be run outside of the Stalker Timeout Only the Beanstalk::Job#ttr applies. If you use this mode, there can be no `before_handlers` for this job.

* `'explicit_delete' => true` Stalker will not automatically delete the Beanstalk job and will not call `Stalker.log_job_end`. 
    It is up to you to handle the state of the job after its been started (mainly doing a job.delete, job.bury, etc.)

* `'no_bury_for_error_handler' => true` Stalker will not bury the Beanstalk job if there is an Exception and there is an `error_handler` set.
    It is up to your Stalker.job's `error_handler` to properly bury, delete or release the job


Original Readme
===============

[Beanstalkd](http://kr.github.com/beanstalkd/) is a fast, lightweight queueing backend inspired by mmemcached.  The [Ruby Beanstalk client](http://beanstalk.rubyforge.org/) is a bit raw, however, so Stalker provides a thin wrapper to make job queueing from your Ruby app easy and fun.

Queueing jobs
-------------

From anywhere in your app:

    require 'stalker'

    Stalker.enqueue('email.send', :to => 'joe@example.com')
    Stalker.enqueue('post.cleanup.all')
    Stalker.enqueue('post.cleanup', :id => post.id)

Working jobs
------------

In a standalone file, typically jobs.rb or worker.rb:

    require 'stalker'
    include Stalker

    job 'email.send' do |args|
      Pony.send(:to => args['to'], :subject => "Hello there")
    end

    job 'post.cleanup.all' do |args|
      Post.all.each do |post|
        enqueue('post.cleanup', :id => post.all)
      end
    end

    job 'post.cleanup' do |args|
      Post.find(args['id']).cleanup
    end

Running
-------

First, make sure you have Beanstalkd installed and running:

    $ sudo port install beanstalkd
    $ beanstalkd

Stalker:

    $ sudo gem install stalker

Now run a worker using the stalk binary:

    $ stalk jobs.rb
    Working 3 jobs: [ email.send post.cleanup.all post.cleanup ]
    Working send.email (email=hello@example.com)
    Finished send.email in 31ms

Stalker will log to stdout as it starts working each job, and then again when the job finishes including the ellapsed time in milliseconds.

Filter to a list of jobs you wish to run with an argument:

    $ stalk jobs.rb post.cleanup.all,post.cleanup
    Working 2 jobs: [ post.cleanup.all post.cleanup ]

In a production environment you may run one or more high-priority workers (limited to short/urgent jobs) and any number of regular workers (working all jobs).  For example, two workers working just the email.send job, and four running all jobs:

    $ for i in 1 2; do stalk jobs.rb email.send > log/urgent-worker.log 2>&1; end
    $ for i in 1 2 3 4; do stalk jobs.rb > log/worker.log 2>&1; end

Error Handling
-------------

If you include an `error` block in your jobs definition, that block will be invoked when a worker encounters an error. You might use this to report errors to an external monitoring service:

    error do |e, job, args|
      Exceptional.handle(e)
    end

Before filter
-------------

If you wish to run a block of code prior to any job:

    before do |job|
      puts "About to work #{job}"
    end

Tidbits
-------

* Jobs are serialized as JSON, so you should stick to strings, integers, arrays, and hashes as arguments to jobs.  e.g. don't pass full Ruby objects - use something like an ActiveRecord/MongoMapper/CouchRest id instead.
* Because there are no class definitions associated with jobs, you can queue jobs from anywhere without needing to include your full app's environment.
* If you need to change the location of your Beanstalk from the default (localhost:11300), set BEANSTALK_URL in your environment, e.g. export BEANSTALK_URL=beanstalk://example.com:11300/
* The stalk binary is just for convenience, you can also run a worker with a straight Ruby command:
    $ ruby -r jobs -e Stalker.work

Running the tests
-----------------

If you wish to hack on Stalker, install these extra gems:

    $ gem install contest mocha

Make sure you have a beanstalkd running, then run the tests:

    $ ruby test/stalker_test.rb

Meta
----

Created by Adam Wiggins

Patches from Jamie Cobbett, Scott Water, Keith Rarick, Mark McGranaghan, Sean Walberg, Adam Pohorecki

Heavily inspired by [Minion](http://github.com/orionz/minion) by Orion Henry

Released under the MIT License: http://www.opensource.org/licenses/mit-license.php

http://github.com/adamwiggins/stalker

`Beanstalk::Job` access by [Robert J. Berger](https://github.com/rberger/stalker)

