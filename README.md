# Developing fabric8-analytics

This section is for those interested in contributing to the development of
fabric8-analytics. Please read through our [glossary](./glossary.md) in
case you are not sure about terms used in the docs.

## fabric8-analytics CI/CD

Please find a documentation on how to setup CI/CD for your repository [here](docs/README-cicd.md).

## Running a Local Instance

### Getting All Repos

### Requirements

Git, and possibly other packages, depending on how you want to run the system
(see below).

### Getting the Code

First of all, clone the `fabric8-analytics-deployment` repo (this one). This includes all
the configuration for running the whole system as well as some helper
scripts and docs.

In order to have a good local development experience, the code repositories
are mounted inside containers, so that changes can be observed live or after
container restart (without image rebuilds).

In order to achieve that, all the individual fabric8-analytics repos have to be
checked out. The helper script `setup.sh` is here to do that. Run `setup.sh -h`
and follow the instructions (most of the time, you'll be fine with running
`setup.sh` with no arguments).

### Running in OpenShift

This is not exactly a local development experience, but if you'd like to test your changes
in OpenShift, there is some [documentation](openshift/README.md) on how to do it.


### Running via docker-compose

Requirements:

* docker
* docker-compose

Then run:

```
$ sudo docker-compose up
```

To get the system up.

Please note that some error messages might be displayed during startup for the
`data-model-importer` module.  Such errors are caused by the Gremlin-https that
takes some time to start before serving any requests. After some time, the
`data-model-importer` will be started properly.

If you want a good development setup (source code mounted inside the
containers, ability to rebuild images using docker-compose), use:

```
$ sudo ./docker-compose.sh up
```

`docker-compose.sh` will effectively mount source code from checked out
fabric8-analytics sub-projects into the containers, so any changes made to the local
checkout will be reflected in the running container. Note, that some
containers (e.g. server) will pick this up interactively, others (e.g. worker)
will need a restart to pick the new code up.

```
$ sudo ./docker-compose.sh build --pull
```

#### Secrets

Some parts (GithubTask, LibrariesIoTask) need credentials
for proper operation. You can provide environment variables in `worker_environment`
in `docker-compose.yml`.

#### Scaling

When running locally via docker-compose, you will likely not need to scale
most of the system components. You may, however, want to run more workers,
if you're running more analyses and want them finished faster. By default,
only a single worker is run, but you can scale it to pretty much any number.
Just run the whole system as described above and then in another terminal
window execute:

```
$ sudo docker-compose scale worker-api=2 worker-ingestion=2
```

This will run additional 2 workers, giving you a total of 4 workers running.
You can use this command repeatedly with different numbers to scale up and
down as necessary.

### Accessing Services, Logs and Other Interesting Stuff

#### Services

When the whole application is started, there are several services you can
access. When running through docker-compose, all of these services will be
bound to `localhost`.

1. fabric8-analytics Server itself - port `32000`
2. fabric8-analytics Jobs API - port `34000`
4. PGWeb (web UI for database) - port `31003`
   PGWeb is only run if you run with `-f docker-compose.debug.yml`
5. Minio S3 - port `33000`

#### Logs

All services log to their stdout/stderr, which makes their logs viewable
through Docker:

* When using docker-compose, all logs are streamed to docker-compose output
and you can view them there in real-time. If you want to see output of a single
container only, use `docker logs -f <container>`, e.g.
`docker logs -f coreapi-server` (the `-f` is optional and it switches on
the "follow" mode).

## Integration tests

Refer to the [integration testing README](https://github.com/fabric8-analytics/fabric8-analytics-common/blob/master/integration-tests/README.md)

## Worker types

Worker, by its design, is a monolith that takes care of all tasks available
in the system. However there is a possibility to let a worker serve only some
particular tasks. This can be accomplished by supplying the following
environment variables (disjoint):

* `WORKER_EXCLUDE_QUEUES` - a comma separated list of regexps describing name
                            of queues that should be *excluded* from worker
                            serving, all others are included
* `WORKER_INCLUDE_QUEUES` - a comma separated list of regexps describing name
                            of queues that should be *included* from worker
                            serving, all others are excluded

This can be especially useful when performing [prioritization or throttling of some particular tasks](http://selinon.readthedocs.io/en/latest/optimization.html#prioritization-of-tasks-and-flows).

## Local deployment

Even though the whole fabric8-analytics is run in Amazon AWS, there are basically
no strict limitations in local deployment.

The following AWS resources are used in the cloud deployment.

* Amazon SQS
* Amazon DynamoDB
* Amazon S3
* Amazon RDS

However, there are used alternatives in the local setup:

* RabbitMQ - an alternative to Amazon SQS
* Local DynamoDB
* Minio S3 - an alternative to Amazon S3
* PostgreSQL - an alternative to Amazon RDS

### Using RabbitMQ instead of Amazon SQS

You can use directly RabbitMQ instead of Amazon SQS. To do so, do **not** provide
`AWS_SQS_ACCESS_KEY_ID` and `AWS_SQS_SECRET_ACCESS_KEY` environment variables.
The RabbitMQ broker should be then running on a host described by
`RABBITMQ_SERVICE_SERVICE_HOST` environment variable (defaults to `coreapi-broker`).

As RabbitMQ scales pretty flawlessly, there shouldn't be any strict downside of
using RabbitMQ instead of Amazon's SQS.

### Using local DynamoDB instead of the hosted one

The local setup is already prepared to work with local DynamoDB.

### Using Minio S3 instead of Amazon S3

To substitute Amazon S3 in local deployment and development setup, there
was chosen [Minio S3](https://github.com/minio/minio) as an alternative. However
Minio S3 is not fully compatible alternative (e.g. it does not support versioning,
nor encryption), but it does not restrict basic functionality whatsoever.

You can find credentials to the Minio S3 web console in `docker-compose.yml` file
(search for `MINIO_ACCESS_KEY` and `MINIO_SECRET_KEY`).

### Using PostgreSQL instead of Amazon RDS

The only difference in using PostgreSQL instead of Amazon's RDS is in
supplying connection string to PostgreSQL/Amazon RDS instance. This connection
string is constructed from the following environment variables:

* `POSTGRESQL_USER` - defaults to `coreapi` in the local setup
* `POSTGRESQL_PASSWORD` - defaults to `coreapi` in the local setup
* `POSTGRESQL_DATABASE` - defaults to `coreapi` in the local
* `PGBOUNCER_SERVICE_HOST` - defaults to `coreapi-pgbouncer` (note that PostgreSQL/Amazon RDS is accessed through PGBouncer, thus naming)
