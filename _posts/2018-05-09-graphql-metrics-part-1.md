---
layout: post
title: Make your own GraphQL metrics dashboard - Part 1
---

Hello and welcome to my multi-part tutorial series on how to build your own GraphQL metrics dashboard for fun! In this series will leverage the Apollo Tracing interface to collected and parse metric data from your GraphQL API. This series is inspired by the incredibly cool Apollo Engine, which uses the tracing extension we'll use to get that awesome performance data. GraphQL has a unique execution pattern that allows us to do some really cool stuff that isn't possible with plain REST out of the box, like query usage on a per field basis. Due to GraphQL's ability to resolve data from multiple sources asynchronously and piece  it together, we can see which areas of our queries are slowing down the overall response and build timelines for each query.

Let's start with a mocked GraphQL server, feel free to use an existing GraphQL server if you want. It'll allow us to generate and inspect that trace data we need for our dashboard later. Here's what it'll look like when you make a request:

```
   Client        Proxy        Mock (or real API)
     |             |             |
 1.  | --- req --> |             |
 2.  |             | --- req --> |
 3.  |             | <-- res --- |
 4.  | <-- res --  |             |
     |             |             |
```

1. A client makes a GraphQL request to the proxy.
2. The proxy pipes the request to the GraphQL server.
3. The proxy takes the response and collects the metric data.
4. The proxy returns the data response to the client.

Later the proxy will send the metric data to our metrics database. Let's jump right in. I use `TypeScript` but feel free to use `Babel` or any other `ES6` tools, I'm mostly going to use `ES6` features in this tutorial series anyways.

```
$ mkdir graphql-metrics
$ cd graphql-metrics
$ yarn init -y
$ yarn add apollo-server-express express request graphql graphql-tools
$ yarn add -D typescript ts-node nodemon
```

Here's the start script you want to add to `package.json`:

```json
"start": "nodemon -e ts -x 'ts-node mock-server.ts'"
```

Let's start that `mock-server.ts` file:

```typescript
import { graphiqlExpress, graphqlExpress } from 'apollo-server-express';
import { spawn } from 'child_process';
import * as express from 'express';
import { readFileSync } from 'fs';
import { addMockFunctionsToSchema, makeExecutableSchema } from 'graphql-tools';
import { resolve } from 'path';

const typeDefs = readFileSync(__dirname + '/schema.graphql', 'utf8');
const schema = makeExecutableSchema({ typeDefs });
// this will send dummy data responses so no need for resolvers
addMockFunctionsToSchema({ schema });

const app = express();

// Start proxy server
const proxy = spawn('ts-node', [__dirname + '/proxy.ts']);
proxy.stdout.pipe(process.stdout);
proxy.stderr.pipe(process.stderr);
process.on('exit', () => proxy.kill());

// Setup regular apollo middleware
app.use('/graphql', express.json(), graphqlExpress({
  schema,
  // tracing enables the performance metrics we'll collect
  // with the proxy to create a dashboard
  tracing: true,
}));

app.use('/graphiql', graphiqlExpress({ endpointURL: '/graphql' }));

app.listen(5000, () => {
  console.log(`GraphQL started http://localhost:5000/graphiql`);
});
```

and a super simple `schema.graphql` for testing, if you don't already have one:

```typescript
type Query {
  me: User
}

type User {
  name: String
  email: String
  age: Int
}
```

If you were to transplant this into an existing apollo-server-express app all you need to do is spawn the proxy server and enable tracing to get started. Awesome so far, let's run `yarn start` and check out the tracing data.

![Tracing output in GraphiQL]({{ "/assets/img/tracing.png" | absolute_url }})

That's interesting! Now we see all this extra stuff under `extensions`, this is the performance trace for this query, tailored for GraphQL. Later we'll filter this out since the client doesn't need it but for now we'll keep it. Now that you've done all that you've done all that finger exercising, [here's the GitHub repo](https://github.com/mpicard/graphql-metrics-proxy/tree/part-1) for the proxy in case you want to copy my tsconfig, etc files. Stay tuned for part two when we dive into the proxy server which will collect and filter this output into postgres using `jsonb`!
