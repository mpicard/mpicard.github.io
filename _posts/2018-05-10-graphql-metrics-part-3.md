---
layout: post
title: Make your own GraphQL metrics dashboard - Part 3
---

Hey there! Welcome back to "DIY Apollo Engine but way less nice"! We've already covered a lot of ground in tutorials [part 1]({% post_url 2018-05-09-graphql-metrics-part-1 %}) and [part 2]({% post_url 2018-05-09-graphql-metrics-part-2 %}) so I highly recommend going back to catch up before continuing, otherwise to recap: we're building a performance monitoring bashboard specifically tailored for `GraphQL` using a proxy and a metrics API with `postgres`.

The next piece of the puzzle is our metrics API. The proxy will `post` tracing data for each query to the metrics API asynchronously. The metrics API will store our tracing data in `postgres` in a few tables and we'll construct some views on top to aggregate performance data. After that we'll expose this aggregate date via graphQL (obviously!) to our dashboard UI. Our architecture is growing:

```
   Client        Proxy        Mock (or real API)
     |             |             |
     | --- req --> |             |
     |             | --- req --> |
     |             | <-- res --- |
     | <-- res --  |
     |             |
     |             |             Metrics API      Dashboard UI
     |             |                 |                 |
     |             | --- metrics --> | --- metrics --> |
     |             |                 |                 |
```

So let's get started! We'll start by initalizing our project with all the packages we'll need:

```
> $ mkdir api
> $ cd api
> $ yarn init -y
> $ yarn add -D typescript dotenv nodemon ts-node @types/{express,node,pg}
> $ yarn add express pg
```

I'll be using `TypeScript` because it's amazing but feel free to use any `es6` tools you want. We'll also use `docker-compose` to manage our `postgres` instance but feel free to run `postgres` if you know/perfer other methods.

```yaml
# docker-compose.yml
version: '3'
services:
  db:
    image: postgres:latest
    ports:
      - 5432:5432
    environment:
      POSTGRES_PASSWORD: postgres
```

We'll use `dotenv` to manage environment variables for postgres:

```bash
# .env
PGHOST='localost'
PGUSER='postgres'
PGDATABASE='postgres'
PGPASSWORD='postgres'
```

In `package.json` we'll add a start script:

```json
"start": "nodemon -e ts -x 'ts-node -r dotenv/config' src/index.ts"
```

Let's start hacking!

```typescript
// src/index.ts
import * as express from 'express';

import { processMetric } from './api';
import { db } from './connectors';
import { Operations, Traces } from './models';

const app = express();

app.use(express.json());

// Collect metric data
app.post('/api/metrics', processMetric);

app.listen(8000, () => {
  console.log('Metrics API started http://localhost:8000/graphiql');

  // Initialize postgres or exit
  db.connect()
    .then(() => db.query(`create extension if not exists "uuid-ossp";`))
    .then(() => Operations.init())
    .then(() => Traces.init())
    .catch(err => {
      console.error("pg error:", err);
      process.exit(1);
    });
});
```

Ok so we'll create our first model, the Operation, which will hold unique graphQL queries using the stipped down `query` string and, if you used a graphQL named operation, that as `name`. If we select from `operation` our data will looks something like this:

```
postgres=# select name, query from operation ;
  name   |                query
---------+----------------------------------------
 MeQuery | query MeQuery { me { name email age } }
         | { me { name } }
         | { me { age } }
         | { fn(a, b) { age } }
         | { fn(a, b) { name email } }
```

```typescript
// src/models/operations.ts
import { db } from '../connectors';
import { Traces } from './traces';

/**
 * Operations store all unique GraphQL queries
 */
export class Operations {

  static init() {
    return db.query(`create table if not exists operation (
                      id    uuid primary key default uuid_generate_v4(),
                      query text unique not null,
                      name  text
                    );`);
  }

  static create({ query, operationName, extensions }) {
    return db
      .query(`insert into operation (query, name) values ($1, $2) returning id`)
      .then(res => res.rows[0].id)
      .then(operationId => {
        // ensure tracing data included
        if (extensions && extensions.tracing) {
          Traces.create(operationId,  extensions.tracing);
        }
      })
      .catch(err => {
        // ignore operation_query_key duplicate queries
        if (err.constraint !== 'operation_query_key') {
          throw err;
        }
      });
  }

}
```

Next is the Trace model which holds most of the metrics themselves like `duration`, `startTime`, etc. as well as all the resolvers as an array in `jsonb`! The `jsonb` data type is really cool because it stores `json` in binary format which is very fast and fully searchable and indexable! We'll levage this to build some views to aggregate query performance!

```typescript
// src/models/traces.ts

export class Traces {

  static init() {
    return db
      .query(`create table if not exists trace (
                id           uuid primary key default uuid_generate_v4(),
                operation_id uuid references operation(id),
                version      smallint not null,
                start_time   timestamp with time zone not null,
                end_time     timestamp with time zone not null,
                duration     integer not null,
                resolvers    jsonb
              );`)
      .then(() => db.query(`create index if not exists trace_operation_id_idx
                              on trace(operation_id);`))
      .then(() => db.query(`create index if not exists trace_resolvers_idx
                              on trace using gin (resolvers);`));
  }

  static create(operationId, tracing) {
    const { version, startTime, endTime, duration } = tracing;
    const resolvers = JSON.stringify(tracing.execution.resolvers);
    const values = [operationId, version, startTime, endTime, duration, resolvers];
    return db.query(`insert into trace (operation_id, version, start_time,
                     end_time, duration, resolvers) values
                     ($1, $2, $3, $4, $5, $6);`, values);
  }

}
```

Wow! Let's try this puppy out! We can go to our `proxy` project and run `yarn start` and also run `yarn start` inside our `api` project. Now nagivate to the proxy [here](http://localhost:4000/graphiql) and make a few queries! Now open `psql`:

```
> $ psql -h localhost -U postgres

postgres=# select * from operation;
```

![Tracing output in GraphiQL]({{ "/assets/img/operations.png" | absolute_url }})

```
postgres=# select * from trace;
```

![Example operation select]({{ "/assets/img/traces.png" | absolute_url }})

Congrats everyone! Thanks for reading this far I really appreciate it! I hope you're learning lots or getting some nice ideas! Maybe you're appalled by my `SQL` or my `TypeScript/JavaScript` so please let me know on the [Issues tracker](https://github.com/mpicard/graphql-metrics-api/issues) or if you want to clone/inspect/fork the repo [here](https://github.com/mpicard/graphql-metrics-api/tree/part-3). Feel free to ask questions you have on any of the issue trackers for the `proxy` or the `api` or <a href="mailto:martin8768@gmail.com">email me</a>. Stay tuned for part 4 when we start building views to aggregate performance data into stuff like how often is this field is used vs other fields on a type, how many requests per minute is my query getting and what's the average response time for your queries. Later on we'll get even more detailed and drill down into each resolver's performance, since that's something graphQL can do!
