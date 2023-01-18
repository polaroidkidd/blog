# Microservices with NodeJS and React (the nodejs bits)
These are my notes (focusing mainly on the backend topics) from [Stephan Grinder's](https://www.udemy.com/course/microservices-with-node-js-and-react/#instructor-1) excellent udemy course [Microservices with Node JS and React](https://www.udemy.com/course/microservices-with-node-js-and-react/). So far my notes cover the first section of his course.


**Author: Daniel Einars**

**Date Published: 05.02.2023**


## 1. Fundamental Ideas Around Microservices
### 1.1 What is a microservice

For the sake of this article, the working definition of a microservice is

 > A service which contains routing, middlewares, business logic and database access to implement one feature of an application.

### 1.2 Data in Microservices

One of the big challenges in microservices is data management. Specifically how we store data in a service, and how that data communicates with other services. 

There's two basic rules when it comes to services and data. Those are

1. Every service gets its own database (if it needs one)
2. No service will ever reach into the database of another service.

This is commonly known as the [Database-Per-Service pattern](https://microservices.io/patterns/data/database-per-service.html). The three main reasons for applying this pattern are

1. Services should be able to run independently of each other. If one service fails, you don't want other services also to fail because of their dependency.
2. Database schema/structure changes should remain isolated to a single service. You don't want to have to change DB access logic in a large number of services only because you changed the schema of a single service
3. Depending on the service, it might run more efficiently with different types of DBs.

### 1.3 Sync Communication Between Services

So, this begs the question...

> How do we then sync data between services?

The two general stratagies to deal with this. These are

1. Synchronous - Services communicate with each other using their respective APIs using direct requests
   * **Pro:** Simple to implement on small scale and uses existing methods
   * **Con:** Introduces dependencies between services, even if it is just between their APIs
   * **Con:** If any inter-service requests fails, overall requests will fail aswell
   * **Con:** Requests which collect data from a number of other services, are only as fast as the slowest request
   * **Con:** Can introduce a web of dependencies, which can spiral out of control
2. Asynchronous - Services communicate with each other using _events_
   * Pro: Services can subscribe to an event bus in order to collect information they need to operate
   * Con: The Eventbus is a single point of failure

### 1.4 Event-Based Communication

In asynchronous, the services which are hooked up to the event bus emit _events_ when ever something happens that another service could be interested in. For example, if you have a user-service, adn a new user signs up, the user service will emit an event detailing that user's data (firstName, lastName, eMail, etc.) and the user management service will listen to that event and create a new entry in the user database.

Using this approach, a services which summarizes data from other services (i.e. a service which shows all orders from a given user) will now be completely independent of other services and requesting data from this service will als no longer be as slow as the slowest service. The downside of this is data duplication as this "summary" service has to keep its own database. Additionally, it is a little harder to wrap your head around, but not impossible.

Now, if you really complain about using extra storage, keep in mind how _cheap_ data storage actually is these days. Seriously, it's a no-brainer.
