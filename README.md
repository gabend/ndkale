# Kale: Distributed task worker from Nextdoor

[![Build Status](https://travis-ci.org/Nextdoor/ndkale.svg?branch=master)](https://travis-ci.org/Nextdoor/ndkale)

Kale is a python task worker library that supports priority queues on Amazon SQS.

We have a blog post on Kale: [http://tmblr.co/ZZM89v1O9x1RI](http://tmblr.co/ZZM89v1O9x1RI)

## How does it work?

![Kale-based Taskworker](https://s3.amazonaws.com/files.corp.nextdoor.com/github_photos/Task+Consumption.png)

Like other distributed task queue system, publishers send task messages to queues and workers fetch messages from queues. For now, Kale supports only Amazon SQS for the queue.

### Publisher

A publisher can be any python program that imports a Kale-based task class and invokes the publish function of this class. For example, if a task class looks like this:

    # tasks.py
    
    class MyTask:
        def run_task(self, arg1, arg2, *args, **kwargs):
            # Do something
            

Then the publisher publishes a task to Amazon SQS, which normally takes 10s miliseconds to return:

    import tasks
    tasks.MyTask.publish(arg1, arg2)
   
The publish() function is a static method of a task class and it has the same signiture as the run\_task() method. A worker process, which may run on a different machine, will pick up the message and execute run\_task() method of the task.

### Worker

![Kale-based Taskworker](https://s3.amazonaws.com/files.corp.nextdoor.com/github_photos/Task+Consumption+Lifetime.png)

A worker process runs an infinite loop. For each iteration, it does the following things:

1. It runs a **queue selection algorithm** (select_queue) to decides which queue to fetch tasks from;
2. It fetches a batch of tasks from a queue (get_messages);
3. It runs tasks one by one in the same batch (run_task);
4. Finish up.
    1. If a task succeeded, it'll be deleted from the queue;
    2. If a task runs too long (exceeding **time_limit** that is a per task property for task SLA) or it fails,
      it'll be put back to the queue and other workers will pick it up in the future (if retry is allowed);
    3. If a batch of tasks runs too long, exceeding **visibility timeout** that is a per queue property for task
      batch SLA, then unfinished tasks will be put back to the queue and other workers will pick them up in the
      future.
5. It exits this iteration and enters next iteration and repeat the above steps.

#### Queue Selection Algorithm

Code: kale/queue_selector.py

A good queue selection algorithm has these requirements:

1. higher priority queues should have more chances to be selected than lower priority queues;
2. it should not starve low priority queues;
3. it should not send too many requests to SQS while retrieving no task, which is waste of
   compute resource and Amazon charges us by the number of requests.
4. it should not wait on empty queues for too long, avoiding waste of compute resources.

We experimented and benchmarked quite a few queue selection algorithms. We end up using an
improved version of lottery algorithm, **ReducedLottery**, which fulfill the above requirements.

ReducedLottery works like this:

        Initialize the lottery pool with all queues
        while lottery pool is not empty:
            Run lottery based on queue priority to get a queue who wins the jackpot
            Short poll SQS to see if the selected queue is empty
            if the selected queue is not empty:
                return queue
            else:
                Remove this queue from the lottery pool
        Reset the lottery pool with all queues
        Return whatever queue who wins the jackpot

The beauty of **ReducedLottery**:

* It prefers higher priority queues, as higher priority queues get more lottery tickets and have
  higher chances to win the jackpot. Thus, requirement 1 is fulfilled.
* It uses randomness to avoid starvation. Lower priority queues still have chance to win the jackpot.
  Thus, requirement 2 is fulfilled.
* If the selected queue is empty, SQS will automatically let task worker long poll on the queue,
  avoiding sending too many requests (short polls). Thus, requirement 3 is fulfilled.
* It excludes known empty queues from the lottery pool. Only when all queues are empty can it returns
  an empty queue. So, it's unlikely to long poll on an empty queue. Thus, requirement 4 is fulfilled.

#### Settings

There are two types of settings, worker config and queue config.

##### Worker config

Settings are specified in settings modules, including AWS confidentials, queues config, queue selection
algorithm, ...

Settings modules are loaded in such order:

* kale.default_settings
* the module specified via KALE\_SETTINGS\_MODULE environment variable

Here's an example

        import os

        AWS_REGION = 'us-west-2'

        #
        # Production settings
        # (use this for prod to talk to Amazon SQS)

        # MESSAGE_QUEUE_USE_PROXY = False
        # AWS_ACCESS_KEY_ID = 'AWS KEY ID'
        # AWS_SECRET_ACCESS_KEY = ''AWS SECRET KEY

        #
        # Development settings
        # (use this for dev to talk to ElasticMQ, which is SQS emulator)

        # Using elasticmq to emulate SQS locally
        MESSAGE_QUEUE_USE_PROXY = True
        MESSAGE_QUEUE_PROXY_PORT = 9324
        MESSAGE_QUEUE_PROXY_HOST = os.getenv('MESSAGE_QUEUE_PROXY_HOST', '0.0.0.0')
        AWS_ACCESS_KEY_ID = 'x'
        AWS_SECRET_ACCESS_KEY = 'x'

        QUEUE_CONFIG = 'taskworker/queue_config.yaml'

        # SQS limits per message size, bytes
        # It can be set anywhere from 1024 bytes (1KB), up to 262144 bytes (256KB).
        # See http://aws.amazon.com/sqs/faqs/
        SQS_TASK_SIZE_LIMIT = 256000

        QUEUE_SELECTOR = 'kale.queue_selector.ReducedLottery'


Settings in the later modules overwrite those in the early-loaded modules.

##### Queue config

All queues and their properties are in a queues config yaml file whose path is specified in the above
settings modules.

Here's an example

        # task SLA: 60/10 = 6 seconds
        high_priority:
            name: high_priority
            priority: 100
            batch_size: 10
            visibility_timeout_sec: 60
            long_poll_time_sec: 1
            num_iterations: 10

        # task SLA: 60 / 10 = 6 seconds
            default:
            name: default
            priority: 40
            batch_size: 10
            visibility_timeout_sec: 60
            long_poll_time_sec: 1
            num_iterations: 5

        # task SLA: 60 / 10 = 6 seconds
        low_priority:
            name: low_priority
            priority: 5
            batch_size: 10
            visibility_timeout_sec: 60
            long_poll_time_sec: 5
            num_iterations: 5

## How to implement a distributed task worker system using Kale

### Install kale
    
From source code
    
    python setup.py install
    
Using pip

    # Put this in requirements.txt
    git+https://github.com/Nextdoor/ndkale.git@dd5c2d29a7497a034104df43479e835a80e2b27d
    
(We'll upload the package to PyPI soon.)

### Example implementation

See code in the example/ directory.
