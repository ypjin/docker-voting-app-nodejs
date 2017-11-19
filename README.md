# Docker Voting App (Node.js version)

This app is inspired by the existing [Example Voting App](https://github.com/dockersamples/example-voting-app)
by [ManoMarks](https://github.com/dockersamples/example-voting-app), head of developer relations at Docker.

The existing app is an excellent demonstration of how Docker can be used to containerize any of the
processes of a modern application regardless of the programming language used and runtime environment
needed for any specific one; it is a nice example of a [polyglot](http://memeagora.blogspot.com/2006/12/polyglot-programming.html) application.

While this is very useful, particularly from a devops perspective, if you are interested in studying the
application itself, this use of multiple languages, even as simple as it is, raises the barrier to comprehension
slightly. In contrast, this version of the voting app is intended to support an introductory
course on Docker and meant to be easier to follow and comprehend for those already familiar with modern
[JavaScript](https://www.javascript.com/) and [Node.js](https://nodejs.org/). In any case the
use of a single programming language should still be a bit easier to follow for other programmers,
especially if they are already familiar with at least one other member of the
[C family of languages](https://en.wikipedia.org/wiki/List_of_C-family_programming_languages).

Modest as the app is in terms of functionality (it only supports casting votes and querying the vote
tally), nevertheless it has been designed and implemented to showcase principles of a relatively
sophisticated, modern [12-Factor App](https://12factor.net/) exploiting Docker.

The app will be used for a course called **Software Containerization with Docker for Developers**, a
[UC Davis Extension](https://extension.ucdavis.edu/online-learning) online program that
will be available through [Coursera](https://www.coursera.org/) in early 2018. However, the app
itself is open source and free for any use in compliance with the terms of the
[MIT License](https://github.com/subfuzion/example-voting-app-nodejs/blob/master/LICENSE).

See the [wiki](https://github.com/subfuzion/docker-ucdavis-coursera/wiki) for details about the course.

## Application Architecture

![Voting app architecture](https://raw.githubusercontent.com/subfuzion/docker-ucdavis-coursera/master/images/voting-app-architecture.png)

The application consists of the following:

 * **Voter** - this is the client application that users use to cast votes.
 * **Vote** - this is the service that provides a REST API for casting votes and
   retrieving vote tallies. The `voter` application is the client that uses
   this API. When a vote is posted to the API, the service pushes it to a queue
   for subsequent, asynchronous processing. When a request is made for vote
   results, the service submits a query to a database where votes are ultimately
   stored by a worker processing the queue.
 * **Worker** - this is a background service that watches a queue for votes and stores
   them in a database. The worker represents a typical service component designed
   to scale as needed to handle various asynchronous processing tasks, typically pulled from a queue,
   so that the main service doesn't get bogged down handling requests.
 * **Queue** - this is a service that allows the vote service to post votes without
   slowing down to do any special processing or waiting for a database to save
   votes. The queue is an in-memory data store (using Redis) that enhances performance
   by not requiring the vote service to wait on the database since database operations
   that involve disk I/O are much slower; consequently, the vote service is ready
   to continue to accept new API requests faster. Redis (and similar tools) are
   typical components of many real-world applications that require message queue,
   publish-subscribe, key-value storage, or caching support.
 * **Database** - this service (using MongoDB) provides structured data storage and query
   support. One or more types of database are typical service components of most
   real world applications.


## Launching the app

The process is intentionally manual for the introduction course to allow gaining
familiarity with various processes. We will build on this until ultimately we have
a fully automated processes that starts the app services in a Docker swarm.

### Create a bridge network

Create a bridge network that will be shared by the services for communication with
each other:

    $ docker network create -d bridge demonet

### Start a MongoDB container

MongoDB is a NoSQL database that we will use for storing votes. There is already
an existing image that we can use:

    $ docker service create --name mongo -p 27017:27017 --network demonet mongo

### Start a Redis container

Redis provides a fast in-memory data store that we will use as a queue. Votes will
be pushed to the queue, where workers will dequeue them asynchronously for saving
to the MongoDB database for subsquent query processing.

Like, MongoDB, there is already an existing image that we can use:

    $ docker service create \
        --name=redis \
        --publish=6379:6379 \
        --network=demonet \
        --health-cmd='[ $(redis-cli ping) = "PONG" ] || exit 1'
        --health-timeout=5s \
        --health-retries=5 \
        --health-interval=5s \
        redis

or as one line:

    $ docker service create --name=redis --publish=6379:6379 --network=demonet --health-cmd='[ $(redis-cli ping) = "PONG" ] || exit 1' --health-timeout=5s --health-retries=5 --health-interval=5s redis

### Start a Vote Worker container

A vote worker pulls votes from the queue and saves them to the database for
subsequent query processing.

You will need to build the image first:

    $ cd worker
    $ docker build -t worker .

Then you can start it:

    $ docker service create --name worker --network demonet worker
    
### Start a Vote API container 

The `vote` service provides the API that clients will use to submit votes and fetch
voting tally results. The vote service receives votes that it then pushes to the
queue (where they will subsequently pulled by workers and saved to the database),
and it also queries the database to tally the votes. 

You will need to build the image first:

    $ cd vote
    $ docker build -t vote .

Then you can start it:

    $ docker service create --name vote -p 3000:3000 --network demonet vote

### Run a Vote client container

The `vote` app is a terminal program that you run to cast votes and fetch the
voting results. We are running this as an interactive container in a terminal,
not a service, but since we created the overlay network with the `--attachable`
flag, we can share the network with the container so that it will be able to
access the vote api by DNS name.

    $ docker run -it --rm --network demonet subfuzion/voter demonet <cmd>

where <cmd> is either `vote` or `results` (if you don't enter any command,
then usage help will be printed to the terminal).

### Run the assessor

The `assessor` is for evaluating the performance of the Voting App
running under Docker. It works by monitoring the logs of each service
for patterns that must be matched to indicate success. The assessor
produces a report when complete or when the evaluation times out. 

See [here](https://github.com/subfuzion/docker-ucdavis-coursera/wiki#final-project)
for instructions on running an assessment for the final project.
