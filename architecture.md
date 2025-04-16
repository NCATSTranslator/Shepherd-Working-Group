# Shepherd Architecture

This document defines the architecture of Shepherd and the technologies used.

## Overall

Shepherd will consist of the following services:

- A web server to receive queries from the ARS, developers, users, etc.
- A relational database to store query state/metadata
- A JSON object store for query and response objects
- A message broker to distribute tasks to all shepherd services
- Operation workers to perform all Translator ARA functions

![Shepherd Architecture](./Shepherd%20Architecture.jpg)

### Web Server

The web server will use FastAPI:

- It is written in Python, so the majority of Translator developers will be able to understand and interact with it
- It has native support for OpenAPI and thus be able to be used in SmartAPI
- It has great support for OpenTelemetry. With a few extra imports and lines of code, it will be fully integrated with translator-otel.
- It is very performant and can handle 1000s of concurrent requests
- It can scale easily in Kubernetes by spinning up new instances

### Relational Database

The relational database will use PostgreSQL:

- It is a tried and true relational database
- It uses SQL, which is a standard language for database interactions
- An [ARAX tool](https://arax.ncats.io/?recent=24) already exists that can be used for monitoring the database
- There are ways to horizontally scale it by sharding, replication, etc.

#### Database Schema

There will be two database tables:

- shepherd_brain:
  - parent_pk: Given by the ARS, is the uuid of an initial query
  - child_pk: Given by the ARS, is the uuid of a query sent to a specific ARA
  - start_time: The timestamp in UTC of when the query was first received
  - stop_time: The timestamp in UTC of when the query was completed
  - submitter: The name of the service that sent the query
  - remote_ip: The IP address of the submitter
  - domain: The url of the receiving Shepherd server
  - hostname: The uuid of the receiving Shepherd server process
  - response_id: The uuid of the response message of the query
  - callback_url: For async queries, given in the initial query, where the response should be POSTed back once completed
  - state: The current state of the query, i.e. Queued, Running, Completed, etc.
  - status: The current status of the query, i.e. Ok, No Results, other errors, etc.
  - description: A short text description containing query and/or error details
- callbacks:
  - child_pk: A foreign key, links back to the child_pk that is stored in shepherd_brain
  - subquery_id: The uuid of a subquery generated usually during the query expansion portion of the lookup operation

### Message Broker

The message broker will use Redis Streams:

- Redis is tried and true
- It is "pull" based, meaning workers are able to grab new tasks when they are ready for them, and a stable connection to Redis is not needed
- It supports consumer groups, meaning there can be multiple operation workers polling the same queue and Redis guarantees that only one worker will get each task
- A tool, Redis Insights, provides an easy way to monitor

### Query and Response JSON Store

The query and response json store will use Redis:

- It is already being used as the message broker, so it makes sense to use the same technology in other areas
- It is an extremely fast lookup store for real-time query and response retrieval
- It supports connection pools, so database connections will not have to compete with the message broker
- It supports a way to "lock" query response values so that only one worker can modify it at a time. This allows for asynchronous subquery responses to all be merged into the same final response object

### Operation Workers

The operation workers will be custom docker images:

- Shared functions for database and task queue interaction can be used to retrieve tasks
- Can be written in whatever programming language a developer prefers, though Python is recommended
- As docker images, workers can easily be scaled within Kubernetes
- We should have good insight into service issues as Kubernetes logs can show which specific workers have needed to be restarted
- As custom images, these workers can support practically any use-case an ARA team may need
