---
layout: post
title: Make your own GraphQL metrics dashboard - Part 5
---

Hey everyone and welcome back to `Make your own GraphQL metrics dashboard`. Here's just a reminder to catch up [part 1]({% post_url 2018-05-09-graphql-metrics-part-1 %}), [part 2]({% post_url 2018-05-09-graphql-metrics-part-2 %}), [part 3]({% post_url 2018-05-10-graphql-metrics-part-3 %}), [part 4]({% post_url 2018-05-12-graphql-metrics-part-4 %}) if you're just joining us.

In this post we're going to try and build a list of GraphQL `Type`s from all the traces then do some math to figure out the usage percentage of each of the fields. This is not possible to do with your standard REST so it's a really cool idea. It allows you to make decisions later about your API like "This field is only used 0.1% of queries, maybe we should just drop it" or "This field is being used in almost all the queries (and possibly in WHERE clauses), let's index it or optimize it's resolution". We'll also look at adding simple routing to the UI to change views between the `Operations` and `Schema` tables based on the URL.

So let's create a new model in the `api` repo:

```typescript
// api/src/models/types.ts
import { db } from '../connectors';

export class Types {
  static init() {
    db.query(`create or replace temp view type as (
                select md5(row(name, field)::text) as hash,
                      name,
                      field,
                      return_type
                from (
                  select r->>'parentType' as name,
                         r->>'fieldName' as field,
                         r->>'returnType' as return_type
                  from trace
                  cross join jsonb_array_elements(resolvers)
                  with ordinality as elements(r, idx)
                  group by 1, 2, 3
                  order by 1, 2, 3
                ) x
              );`);
  }
}
```

So let's break down this view a little, in the inner subquery we use cross join jsonb_array_elements(resolvers) from trace, thereby creating a row for each resolver object inside resolvers (json array). We add ordinality

```sql
select r->>'parentType' as name,
       r->>'fieldName' as field,
       r->>'returnType' as return_type
from trace
cross join jsonb_array_elements(resolvers)
with ordinality as elements(r, idx)
group by 1, 2, 3
order by 1, 2, 3;

 name  | field  | return_type
-------+--------+-------------
 Query | fn     | User
 Query | me     | User
 User  | age    | Int
 User  | email  | String
 User  | name   | String
 User  | parent | User
```

Ok but I can already tell `react` is going to complain about not having a `key` prop but this is a derived view so let's just make a simple hash for the `key` prop. We can assume that name and field will always be a unique pair since it's grouped by.

```sql
select md5(row(name, field)::text) as id,
       name,
       field,
       return_type
from (
  select r->>'parentType' as name,
         r->>'fieldName' as field,
         r->>'returnType' as return_type
  from trace
  cross join jsonb_array_elements(resolvers)
  with ordinality as elements(r, idx)
  group by 1, 2, 3
  order by 1, 2, 3
) x;

               id                 | name  | field  | return_type
----------------------------------+-------+--------+-------------
 7a8bf89734f62dee77063596fe8caac1 | Query | fn     | User
 09cc45d11bdd6877bfddb0465fcb3d12 | Query | me     | User
 ff428b6b6708d3fcdbb453d6117be2a3 | User  | age    | Int
 95331ca259e940d1692a20b50b1fe746 | User  | email  | String
 36c67cfe4fd186c2fa98ca431069cb56 | User  | name   | String
 888a5fba1f296404df873c2f0549c7b5 | User  | parent | Users
```

Amazing! I probably won't use this for lookup so looks good enough to me. Wrap that query in a view and it's good to go. The next step is to add a select all method to the model:

```typescript
// api/src/models/types.ts

  // ...
  static allTypes() {
    return db.query(`select * from type;`).then(res => res.rows);
  }
}
```

Then add it to our resolvers and schema:

```typescript
// api/src/resolvers.ts

// ...
  Query: {
    allOperations: () => Operations.allOperations(),
    allTypes: () => Types.allTypes(), // <- newly added
    operation: (_, { id }) => Operations.get(id),
    trace: (_, { id }) => Traces.get(id)
  },
```

```graphql
# api/src/schema.graphql

type Query {
  allOperations: [Operation]
  operation(id: ID!): Operation
  trace(id: ID!): Trace
  allTypes: [Type]
}

type Type {
  id: ID
  name: String
  field: String
  returnType: String
}
```

We'll have to call the init script from the index as well:

```typescript
// ...
import { Operations, Traces, Types } from './models';
// ...
app.listen(8000, () => {
  console.log('API started http://localhost:8000/graphiql');
  db
    .connect()
    .then(() => db.query(`create extension if not exists "uuid-ossp";`))
    .then(() => Operations.init())
    .then(() => Traces.init())
    .then(() => Types.init()) // <- newly added
    .catch(err => {
      console.error('pg error:', err);
      process.exit(1);
    });
});
```

Now if we go back to the UI, we'll realize that we could do simple component switch but I want to render components based off the URL. So I like to use [this method](https://hackernoon.com/routing-in-react-the-uncomplicated-way-b2c5ffaee997) for routing. We'll do ours a little differently, start by installing [history](https://www.npmjs.com/package/history) with `yarn add history`, the create a new file `ui/src/routes.js`:

```javascript
// ui/src/routes.js
import Operations from './components/operations/Operations';
import Schema from './components/schema/Schema';

const routes = {
  '/': Operations,
  '/schema': Schema
};

export default routes;
```

This object will map URLs into components. Then we'll create a little 404 page component:

```javascript
// ui/src/pages/NotFound.js
import React from 'react';

const NotFound = () => <p>404 - Not Found</p>;

export default NotFound;
```

Now let's create a `history` module to handle browswer history, routing and change detection.

```javascript
// ui/src/history.js
import createHistory from 'history/createBrowserHistory';

// we might use universal rendering so check if browser or server
const isBrowser = typeof window !== 'undefined';

const history = isBrowser ? createHistory() : {};

const { location } = history;

const onChangeListeners = [];

function push(pathname) {
  // push a new path into history
  window.history.pushState({}, '', pathname);
  // call all the callbacks
  onChangeListeners.forEach(cb => cb(pathname));
}

function onChange(cb) {
  // add a callback
  onChangeListeners.push(cb);
}

export default { history, location, push, onChange };
```

Now we'll have to refactor our App component into a class and add the router:

```javascript
// ui/src/components/app/App.js
import './App.css';

import PropTypes from 'prop-types';
import React from 'react';

import history from '../../history';
import routes from '../../routes';
import NotFound from '../pages/NotFound';
import Sidebar from '../sidebar/Sidebar';

class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      pathname: props.pathname
    };
  }

  // use prop-types to validate pathname is one of the routes in routes
  static propTypes = {
    pathname: PropTypes.oneOf(Object.keys(routes)).isRequired
  };

  componentDidMount() {
    // update App state when path changes
    history.onChange(pathname => this.setState({ pathname }));
  }

  render() {
    // get the component from routes or display 404 page
    const { pathname } = this.state;
    const Outlet = routes[pathname] || NotFound;

    return (
      <div className="App">
        <header className="App__header">
          <h1 className="App__title">GraphQL Metrics</h1>
        </header>
        <Sidebar />
        <Outlet />
      </div>
    );
  }
}

export default App;
```

Now we'll create a component that's a glorified anchor to push new pathnames into history:

```javascript
// ui/src/components/link/Link.js
import './Link.css';

import React from 'react';

import history from '../../history';

class Link extends React.Component {
  constructor(props) {
    super(props);
    // we'll use this to style the active route anchor
    this.state = { active: history.location.pathname === props.href };
  }

  // update the active state when path changes
  componentDidMount() {
    history.onChange(pathname =>
      this.setState({ active: pathname === this.props.href })
    );
  }

  // self explanatory
  onClick = e => {
    const { href } = this.props;
    const aNewTab = e.metaKey || e.ctrlKey;
    const anExternalLink = href.startsWith('http');

    if (!aNewTab && !anExternalLink) {
      e.preventDefault();
      history.push(href);
    }
  };

  // react people is this the best way to do className like this?
  // Leave a comment on GitHub I'd like to know
  render() {
    const { href, children } = this.props;
    const className = 'Link' + (this.state.active ? ' active' : '');
    return (
      <a className="Link {className}" href={href} onClick={this.onClick}>
        {children}
      </a>
    );
  }
}

export default Link;
```

```css
/* Link.css */
a.Link {
  color: #fffbfc;
}
a.Link.active {
  text-decoration: none;
  color: #ef6461;
}
```

Ok so that was a bit of code to get router going but compared to `react-router`, I feel like we're doing pretty good for a few lines of code:

* Change view based on path
* Subscribe to path changes and get current path easily
* Change styles based on current path

We're now ready to keep going with our dashboard and make a `Schema` component to display that `view` we created earlier.

```javascript
// ui/src/components/schema/Schema.js
import gql from 'graphql-tag';
import React from 'react';
import { Query } from 'react-apollo';

const allTypes = gql`
  {
    allTypes {
      id
      name
      field
      returnType
    }
  }
`;

const Schema = () => (
  <table>
    <thead>
      <tr>
        <th>Type</th>
        <th>Field</th>
        <th>Return Type</th>
      </tr>
    </thead>
    <tbody>
      <SchemaTypes />
    </tbody>
  </table>
);

const SchemaTypes = () => (
  <Query query={allTypes}>
    {({ loading, error, data }) => {
      if (loading) return rowSpan('Loading...');
      if (error) return rowSpan(error.message);

      return data.allTypes.map(SchemaType);
    }}
  </Query>
);

const SchemaType = ({ id, name, field, returnType }) => (
  <tr key={id}>
    <td>{name}</td>
    <td>{field}</td>
    <td>{returnType}</td>
  </tr>
);

const rowSpan = content => (
  <tr>
    <td colSpan="3">{content}</td>
  </tr>
);

export default Schema;
```

Awesome work everyone! If we checkout the page (hopefully we've been checking on it now and again for issues) We should now see something like this:

![Schema table view]({{ "/assets/img/schema-types.png" | absolute_url }})
