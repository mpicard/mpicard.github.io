---
layout: post
title: Make your own GraphQL metrics dashboard - Part 2
---

Hi and welcome back to make your own graphQL performance dashboard! If you missed [part 1]({% post_url 2018-05-09-graphql-metrics-part-1 %}) you should go there to catch up first.

Let's create our awesome `proxy` now! As explained in part 1, the `proxy` will take requests and pipe them to the graphql server, parse the response and send the client a stripped down response. The client won't need the extra data in the tracing body.

```typescript
import { Agent } from 'http';
import * as express from 'express';
import * as request from 'request';

const app = express();

// create a default request to the original graphQL server
const origin = request.defaults({
  baseUrl: 'http://localhost:5000/',
  agent: new Agent({ keepAlive: true })
});

// create a default request to metrics api server
const api = request.defaults({
  baseUrl: 'http://localhost:8000/api/',
  agent: new Agent({ keepAlive: true })
});

// catch all /graphql requests with json body parser
app.use('/graphql', express.json(), (req, res, next) => {
  // we'll save the query string and (optional) operationName
  // build a similar request to the incoming request
  let query, operationName;

  if (req.method === 'POST') {
    query = req.body.query;
    operationName = req.body.operationName;
  } else if (req.method === 'GET') {
    query = req.query.query;
    operationName = req.query.operationName;
  }

  const options = {
    uri: req.originalUrl,
    method: req.method,
    headers: req.headers,
    json: req.body
  };
  origin(options, (err, queryRes, body) => {
    // relay status code to client
    res.status(queryRes.statusCode);

    if (queryRes.statusCode > 400) {
      // relay error body/message to client
      res.send(body).end();
    } else {
      // now we have the GraphQL response with the extensions metrics
      const { data, errors, extensions } = body;

      // send back response to client with data and/or errors
      res.json({ data, errors });

      if (query) collectMetrics(query, operationName, extensions, errors);
    }
  });
});

// pipe all other requests (like graphiql) to original
app.all('*', (req, res) => {
  const options = { uri: req.path, qs: req.query };
  req
    .pipe(origin(options))
    .pipe(res);
});

app.listen(4000, () => {
  console.log('Proxy started http://localhost:4000/graphiql');
});

/**
 * Collect query metadata and post to metrics api.
 *
 * @param query GraphQL query string.
 * @param operationName (optional)
 * @param extensions object with tracing and/or caching metadata
 * @param errors array of Graphql errors
 */
function collectMetrics(query, operationName, extensions, errors) {
  // clean up query to remove sensitive data from parameters
  query = query.replace(/(\:.*?)(?=\,)|(\:.*?)(?=\))/g, '');
  // remove redundant whitespace
  query = query.replace(/\s+/g, ' ').trim();

  console.log(query, operationName, extensions, errors);
}
```

Now if we start the server using `yarn start` and navigate to the [proxy's graphiQL](http://localhost:4000/graphiql) instead of the mock server, we should be able to make a query and not get any of the `extensions` object in the response! Meanwhile, in the console you should see the query, operation name, extensions object and, if any, errors. Very cool! Join us next time in part 3 when we take all that data and put it into a postgres database using `jsonb` and makes some queries!
