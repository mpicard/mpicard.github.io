---
layout: post
title: Make your own GraphQL metrics dashboard - Part 4
---

![Operations table UI]({{ "/assets/img/dash-1.png" | absolute_url }})

Hello and welome back to 'Make your own Graphql metrics dashboard'. If you missed [part 1]({% post_url 2018-05-09-graphql-metrics-part-1 %}), [part 2]({% post_url 2018-05-09-graphql-metrics-part-2 %}) or [part 3]({% post_url 2018-05-10-graphql-metrics-part-3 %}) head there now first.

In case you've never used `docker-compose` before and received an error last time like "unable to connect to port :5432" error you can start your `postgres` instance with the command `docker-compose up -d` which will run in the background on port 5432.

In this part, we'll create a new queries to aggregate each `Operation`'s average duration (aka response time) and average requests per minute, very cool! Then we'll add a `GraphQL` API so that we can start making a basic bashboard UI with `react` (link) and `react-apollo` (link). I was never good at statictics so here goes:

Open `path-to-project/api` and add some packages:

```
> $ yarn add apollo-server-express graphql-tools
```

I'm going to assume some basic knowledge of GraphQL since you either already have at least one GraphQL service in which you probably wanted to monitor. If you're not sure about it then head over [here](http://graphql.com/) to the official website for a great explanation first. Now, create a file called `src/schema.ts`:

```typescript
import { readFileSync } from 'fs';
import { makeExecutableSchema } from 'graphql-tools';

import { resolvers } from './resolvers';

const typeDefs = readFileSync(__dirname + '/schema.graphql', 'utf8');
export const schema = makeExecutableSchema({ typeDefs, resolvers });
```

and also create `src/schema.graphql`, this will be the API our dashboard will query to get that performance data. It'll be simple enough for now but later we'll add more fields:

```graphql
type Query {
  allOperations: [Operation]
  operation(id: ID!): Operation
  trace(id: ID!): Trace
}

type Operation {
  id: ID
  name: String
  query: String
  traces: [Trace]
}

type Trace {
  id: ID
  version: Int
  startTime: String
  endTime: String
  duration: Int
}
```

Let's add a package I forgot earlier, `cors`: `yarn add cors`. Now in the `api/src/index.ts`:

```typescript
import { graphiqlExpress, graphqlExpress } from 'apollo-server-express';
import * as express from 'express';
import * as cors from 'cors';

import { processMetric } from './api';
import { db } from './connectors';
import { Operations, Traces } from './models';
import { schema } from './schema';

const app = express();

app.use(cors()); // add cors

app.use(express.json());

app.post('/api/metrics', processMetric);

app.use('/graphql', graphqlExpress({ schema }));

app.use('/graphiql', graphiqlExpress({ endpointURL: '/graphql' }));

app.listen(8000, () => {
  console.log('API started http://localhost:8000/graphiql');
  db
    .connect()
    .then(() => db.query(`create extension if not exists "uuid-ossp";`))
    .then(() => Operations.init())
    .then(() => Traces.init())
    .catch(err => {
      console.error("pg err:", err);
      process.exit(1);
    });
});
```

And finally we'll wire up our resolvers to our models in `src/resolvers/ts`:

```typescript
import { Operations, Traces } from './models';

export const resolvers = {
  Query: {
    allOperations: () => Operations.allOperations(),
    operation: (_, { id }) => Operations.get(id),
    trace: (_, { id }) => Traces.get(id),
  },
  Operation: {
    traces: ({ id }) => Traces.forOperation(id),
  },
  Trace: {
    startTime: trace => trace.start_time,
    endTime: trace => trace.end_time
  },
}
```

Wow! I love GraphQL, so beautiful :)

Now if we start our server (`yarn start`) we can go to (GraphiQL)[http://localhost:8000/graphiql] and try the following query:

```graphql
{
  allOperations {
    id
    name
    query
    traces {
      startTime
      endTime
      duration
    }
  }
}
```

We should get a bunch of data, if you only get an empty array you probably haven't generated any traces. So if that's the case we'll return to our `proxy` folder and run `yarn start`, then open [GraphiQL](http://localhost:4000/graphiql) and make a few dozen queries and try removing/adding fields to get a few different `Operation`s registered and lots of `Trace`s. Going back to the API and run the query again, you should see the `Operation`s and their `Trace`s now. If you get an error or something else please open an [Issue](https://github.com/mpicard/graphql-metrics-api/issues).

![allOperations query results]({{ "/assets/img/all-operations.png" | absolute_url }})

Now comes the _fun_ part! in `src/models/operations.ts`:

```typescript
export class Operations {

  // ...

  // let's do the easy one first
  // lets take the averge duration from all the traces for
  // a particular operation:
  static avgDuration(operationId) {
    return db
      .query(`select round(avg(duration)) from trace where operation_id = $1`, [operationId])
      .then(res => res.rows[0].round)
      .then(res => res ? res : 0);
  }

  // this one is a little more complex
  // the average request per minute
  // the inner query we truncate the date to
  // the minute and do a count over partition
  // which means count all the rows in each minute
  // now we have the rpm for each minute
  // now we avg that rpm for the average rpm
  static avgRpm(operationId) {
  return db
    .query(`select round(avg(rpm), 2) from (select distinct
            date_trunc('minute', start_time), count(*) over (
            partition by date_trunc('minute', start_time)) as rpm
            from trace where operation_id = $1) as avg`, [operationId])
    .then(res => res.rows[0].round)
    .then(res => res ? res : 0);
  }

}
```

Later in the series we'll look at narrowing the time window for some queries so that we aren't taking the average over the entire time range available, so we can do something like show me the average performance over the past hour, or day, week etc. But for now this is pretty cool I think. Let's add those fields to the schema and resolvers:

```graphql
# ...
type Operation {
  id: ID
  name: String
  query: String
  traces: [Trace]
  avgDuration: Float
  avgRpm: Float
}
# ...
```

```typescript
// ...
  Operation: {
    traces: ({ id }) => Traces.forOperation(id),
    avgDuration: ({ id }) => Operations.avgDuration(id),
    avgRpm: ({ id }) => Operations.avgRpm(id)
  },
// ...
```

Let's test that sucker out:

![allOperations avgRpm and avgDuration]({{ "/assets/img/operations-avg-metrics.png" | absolute_url }})

Very cool, so we've got a (very :P) basic monitoring tool in place let's do what we all came here for: Graphs! Tables! Volume charts! Haha ok ok slow down. We'll start simple and build up from there. Now, I'm an Angular developer by trade, all the companies I've worked for happen to use AngularJS and Angular but to make this more accessible I've learnt a bit of React. I figured most people are using React, especially the GraphQL crowed it seems. So please ignore my really noob React, hopefully it will give you the queries and ideas you need to go build something cool as well if you want!

If you don't know React then I'd suggest following along anyways and you can map my react components into whatever you fancy, using the same queries. So let's dive right in let's `create-react-app ui`. Then we have to add the `Apollo` (link) client stuff: `yarn add apollo-boost graphql graphql-tag react-apollo` and now start the webpack server: `yarn start` and let's get cracking:

```javascript
// index.js
import './styles';

import ApolloClient from 'apollo-boost';
import React from 'react';
import { ApolloProvider } from 'react-apollo';
import ReactDOM from 'react-dom';

import App from './app/App';
import registerServiceWorker from './registerServiceWorker';

const client = new ApolloClient({
  uri: 'http://localhost:8000/graphql'
});

ReactDOM.render(
  <ApolloProvider client={client}>
    <App />
  </ApolloProvider>
  , document.getElementById('root')
);

registerServiceWorker();
```

I think this ApolloProvider is called a higher order component?

```javascript
// App.js
import './App.css';

import React from 'react';

import Operations from '../operations/Operations';
import Sidebar from '../sidebar/Sidebar';

const App = () => (
  <div className="App">
    <header className="App__header">
      <h1 className="App__title">GraphQL Metrics</h1>
    </header>
    <Sidebar />
    <Operations />
  </div>
)

export default App;
```

The `Sidebar` is optional for now since we're not going to do any routing right now. By the way, I will include all the css files in the repo link if you want to use mine.

```javascript
// Sidebar.js
import './Sidebar.css';

import React from 'react';

const Sidebar = () => (
  <nav className="Sidebar">
    <ul>
      <li>
        <a className="active">
          Operations
        </a>
      </li>
    </ul>
  </nav>
)

export default Sidebar;
```

```javascript
// Opereations.js
import './Operations.css';

import gql from 'graphql-tag';
import React from 'react';
import { Query } from 'react-apollo';

import { prettyDuration, prettyNumber } from '../utils';

const allOperations = gql`{
  allOperations {
    id name query avgDuration avgRpm
  }
}`;

const OperationRows = () => (
  <Query query={allOperations}>
    {({ loading, error, data }) => {
      if (loading) return <tr><td>Loading...</td></tr>
      if (error) return <tr><td>Error: {error}</td></tr>

      return data.allOperations.map(op => (
        <tr key={op.id}>
          <td>{op.name ? op.name : op.query}</td>
          <td>{prettyNumber(op.avgRpm)}</td>
          <td>{prettyDuration(op.avgDuration)}</td>
        </tr>
      ))
    }}
  </Query>
)

const Operations = () => (
  <div className="Operations">
    <table>
      <thead>
        <tr>
          <th>Operation</th>
          <th>Average RPM</th>
          <th>Average Duration</th>
        </tr>
      </thead>
      <tbody>
        <OperationRows />
      </tbody>
    </table>
  </div>
)

export default Operations
```

We'll create a few helper functions to format our numbers a little better so `prettyNumber` would take 12653 and return 12.6K. `prettuDuration` is very similar but takes nanoseconds and rounds them up to microseconds, milliseconds, etc.

```javascript
// utils.js

// 12495 => 12.4K
export function prettyNumber(value, digits = 1) {
  const units = ['K', 'M', 'B'];

  let decimal;

  for (let i = units.length - 1; i >= 0; i--) {
    decimal = Math.pow(1000, i + 1);
    if (value <= -decimal || value >= decimal) {
      return +(value / decimal).toFixed(digits) + units[i];
    }
  }
  return value.toFixed(digits);
}

// 53321424 => 53.3ms
export function prettyDuration(ns, digits = 1) {
  const units = ['μs', 'ms', 's'];

  let decimal;

  for (let i = units.length - 1; i >= 0; i--) {
    decimal = Math.pow(1000, i + 1);
    if (ns <= -decimal || ns >= decimal) {
      return +(ns / decimal).toFixed(digits) + units[i];
    }
  }
  return ns + 'ns';
}
```

![Operations table UI]({{ "/assets/img/dash-2.png" | absolute_url }})

Tada! Thanks everyone for following along so far. If you have any questions or comments please go to the [Issues]() and leave something there. I'll also provide the full repo so you can use my stylesheets if you want or make your own. Next time we'll add a few more metric calculations and views centered around more fine grained details in each GraphQL resolver to pin point slow backends.
