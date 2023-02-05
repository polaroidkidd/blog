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


## 2. A Mini-Microservice App

In Section two we'll be building a very simple application with a few microservices.

**Note:** The services and logic written in this section will be mostly done by hand. A lot of this is abstracted away in later sections. This just serves the purpose of getting my hands dirty with microservices and getting a basic understanding of the nitty-gritty details. Do _not_ use the code in this section as a template for future projects. There's better templates out there, sich as [this one](https://docs.microservice-stack.com/introduction/). 

The basic outline of this application is a type of blog, where users can create posts/articles and comment on these.

Since I've already compelted the basic setup of the post and comment services at this point, I'm going to skip these sections, but you can check out the code [here](https://github.com/polaroidkidd/microservices-with-node-js-and-react/tree/af446fa26e628cf2557bf7d3b3c06c53cf1e0f37). My code diverges from the cours' setup at this point because I wanted to run all of it inside  monorepo using typescript and I wanted to make use of [zod](https://zod.dev/). At this point we created two services (posts and comments) which allow you to create and read posts and comments. There's also a small react frontend, which I've also omitted from my notes here since it isn't my focus of learning in this course.

### 2.1 Request Minimization Strategies

**Note:** If you want to see the finished product of this chapter, head on over to my [repo](https://github.com/polaroidkidd/microservices-with-node-js-and-react/tree/f8e66b957fa7d333be2235ffe6ab7ef54ef2ae08),

At this point in the application, if we want to load multiple posts, we also have to request all comments related to those posts separately. This is not something which scales well. We have the option to create a dependency from the posts service to the comments service, but that's the _bad_ approach. Instead, we'll be implementing an asynchronous solution by building a `Query Service`.

Ok, so just a few quick notes on this chapter. 

The `CREATE POST` and the `CREATE COMMENT` posts now forward their calls to the event buss like so.

The only thing we changed here is the data structure. So instead of sending raw `CREATE POST` events, we defined that the event bus accepts the following datastructure.

```typescript
import z from "zod";

export const EventSchema = z.object({
  type: z.string().regex(/Created$/),
  data: z.any(),
});

export type IEventSchema = z.infer<typeof EventSchema>;

```

Currently, events must end in `Created`, but the data can be pretty much anything. I say "currently" because this we might want to have the event bus handle different types of events later. It's just a precaution on my end that I don't accidentally start firing events into the event bus which have no business being there.

Once a Post or Comment service gets a request to create a post or a comment, we forward the updated post/comment to the event bus like so

```typescript

...
      axios.post(`${ServiceEventEndpoints.EVENT_BUS}`, {
        type: Events.PostCreated,
        data: post,
      });
...
```


Note that we do not wait for the response from the event bus inside the request, since we don't really care at this point if the eventbus crashed or not because the post/comment is already created independently of that.

```typescript

/**
 * CREATE POST ROUTE
 */
app.post(
  ROUTES.POSTS,
  async (req: Request<{}, {}, IPost>, res: Response<IPost | ZodError>) => {
    const id = randomBytes(4).toString("hex");

    const { body } = req;

    try {
      // parse request
      const parsedBody = IApiPostSchema.parse(body);
      const post = { id, title: parsedBody.title };

      // parse response
      IPostSchema.parse(post);
      posts[post.id] = post;

      /**
       * !!! IMPORTANT BIT HERE !!!
       * 
       * Shoot off the event to the event bus here
       */
      axios.post(`${ServiceEventEndpoints.EVENT_BUS}`, {
        type: Events.PostCreated,
        data: post,
      });
      /**
       * Nothing more to see here.
       */

      res.status(201).send(post);
    } catch (e) {
      res.status(422).send(e as ZodError);
    }
  }
);

```

As previously mentioned, the event bus does nothing more than forwarding incoming events to all services, including the ones which just sent it the event. As such the entire event bus is very small.

```typescript
import axios from "axios";
import bodyParser from "body-parser";
import cors from "cors";
import type { Request, Response } from "express";
import express from "express";
import type { ZodError } from "zod";

import type { IPost } from "@ms/posts/src/post.zod";

import { ServiceEventEndpoints } from "./constants";
import type { IEventSchema } from "./events.zod";
import { EventSchema } from "./events.zod";

enum Routes {
  EVENTS = "/events",
}

const app = express();
app.use(bodyParser.json());

app.use(cors({ origin: "http://localhost:3000" }));

app.post(
  Routes.EVENTS,
  async (
    req: Request<{}, {}, IEventSchema>,
    res: Response<IPost | ZodError>
  ) => {
    try {
      const parsedEvent = EventSchema.parse(req.body);
      /**
       * !!! IMPORTANT BIT HERE !!!
       *
       * Shoot off the event to the event to all services again, including the new query service
       */
      await axios.post(ServiceEventEndpoints.POSTS, parsedEvent);
      await axios.post(ServiceEventEndpoints.COMMENTS, parsedEvent);
      await axios.post(ServiceEventEndpoints.QUERY, parsedEvent);
      /**
       * Nothing more to see here.
       */
      res.status(200);
      res.send();
    } catch (e) {
      console.error(e);
      res.status(422).send(e as ZodError);
    }
  }
);

app.listen(4005, () => {
  console.log('Service "Eventbus" is listening on 4005');
});

```

In this instance I would argue that I _do_ care if the event succeeds or not, which is why we await the posts and fail if one of them fails. This way I can see at which microservice the post fails. In reality, I don't think I'd care though, since the event bus has to continue delivering events to other services regardless if a single post fails or not.

On to the new Query Service!

> Why are we building this in the first place again?!


Well, currently when the client makes a request, he first requests the available posts from the post service, and then requests all comments from the comment service for that post. This is OK if you have 2 or three posts, but if you have 1000 posts, each with 100 comments... well.. things can get out of hand. This is why we create a Query Service which keeps track of all Posts and the associated comments. When we now request all posts, we can fire a single query to the Query Service and get a complete picture.

Since the Event Bus forwards all `CREATE POST` and `CREATE COMMENT` events to the query service as well, it can keep track of this data.

The schema should look like this

```typescript
import { z } from "zod";

import { CommentSchema } from "@ms/comments/src/comments.zod";
import { IPostSchema } from "@ms/posts/src/post.zod";

export const QueryPostSchema = IPostSchema.extend({
  comments: z.array(CommentSchema),
});

export type IQueryPostSchema = z.infer<typeof QueryPostSchema>;

export const QuerySchema = z.record(z.string().min(1), QueryPostSchema);
export type IQuerySchema = z.infer<typeof QuerySchema>;


// example
type Query = Record<string, {
  postId: string;
  title: string;
  comments: Array<{
    postId: string,
    commentId: string,
    content
  }>
}>
```

The Query service, (as well as the Comment and Post Service) has to implement the `/events` route to receive incoming events and react to them. Currently only the Query Service reacts to them. 

```typescript
// in memory DB
const posts: IQuerySchema = {};


/**
 * Handle Events
 */
app.post(
  ROUTES.EVENTS,
  async (req: Request<IPost | IComment>, res: Response) => {
    const { body } = req;
    try {
      // Check which type of event it is and update the "in memory db" accordingly
      if (body.type === Events.PostCreated) {
        const { title, id } = body.data as IPost;
        posts[id] = { id, title, comments: [] };
      }

      if (body.type === Events.CommentCreated) {
        const { id, content, postId } = body.data as IComment;
        posts[postId].comments = [
          ...posts[postId].comments,
          { id, content, postId },
        ];
      }

      // respond that the db has been updated correctly
      res.status(200);
      res.send();
    } catch (e) {
      console.error(e);
      res.status(422).send(e as ZodError);
    }
  }
);
```

Now, the only thing we need to do when a user wants to see all posts and comments, is provide them with the `post` object. Here's the appropriate route for that.

```typescript
/**
 * Get All Posts
 */
app.get(ROUTES.POSTS, async (req: Request, res: Response<IQuerySchema>) => {
  res.send(posts);
});
```
