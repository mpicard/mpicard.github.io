---
layout: post
title: Make your own GraphQL metrics dashboard - Part 5
---

Hey everyone and welcome back to `Make your own GraphQL metrics dashboard`. Here's just a reminder to catch up [part 1]({% post_url 2018-05-09-graphql-metrics-part-1 %}), [part 2]({% post_url 2018-05-09-graphql-metrics-part-2 %}), [part 3]({% post_url 2018-05-10-graphql-metrics-part-3 %}), [part 4]({% post_url 2018-05-12-graphql-metrics-part-4 %}) if you're just joining us.

In this post we're going to try and build a list of GraphQL `Type`s from all the traces then do some math to figure out the usage percentage of each of the fields. This is not possible to do with your standard REST so it's a really cool idea. It allows you to make decisions later about your API like "This field is only used 0.1% of queries, maybe we should just drop it" or "This field is being used in almost all the queries (and possibly in WHERE clauses), let's index it or optimize it's resolution". We'll also look at adding simple routing to the UI to change views between the `Operations` and `Schema` tables based on the URL.

So let's create a few new models in the `api` repo:

```typescript
// api/src/models/resolvers.ts
import { db } from '../connectors';

export class Resolvers {
  static init() {
    db.query(`
      create or replace temp view resolver as
      select trace.id as trace_id, operation_id, x.*
      from
        trace,
        jsonb_array_elements(resolvers) as resolver,
        jsonb_to_record(resolver) as x(
          path text[],
          duration int,
          "fieldName" text,
          "parentType" text,
          "returnType" text,
          "startOffset" int
        )`);
  }
}
```

So this is our first `view`. If you've never used a `sql` view before it's basically a shortcut query, every time we `select ... from resolver` it will run the query above first. Is it the best for performance? Probably not in the long run but rememeber, this is just for fun and learning. I will most likely be posting updates and optimizations as we go along. `view`s make our lives easier and I'll take that option first! This view will be crucial to building more complex queries and views for metrics! It takes all the `trace`s inserted and splits each `resolver` in `resolvers` into rows based on the schema of `x()`. This means we can be a little more flexible if the Apollo Tracing format changes. We need to use double quotes whenever fields are mixed cased in `postgres`, slightly annoying but less so since it will map nicely to Javascript.

We'll create another model that will list all the unique graphQL fields we have:

```typescript
// api/src/models/types.ts
import { db } from '../connectors';

export class Types {
  static allTypes() {
    return db
      .query(
        `select md5(row("parentType", "fieldName")::text) as key,
                "parentType",
                "fieldName",
                "returnType"
          from resolver
          group by 2, 3, 4;`
      )
      .then(res => res.rows);
  }
}
```

So you might be wondering why I created a md5 hash, well in `react` we all know that it likes to have a `key` prop when mapping components so this is a cheap way to have something to use as `key`. I won't call the column `id` because it isn't a primary key and we won't be doing lookups with it. Add it to our `api` schema and resolvers:

```graphql
# api/src/schema.graphql

type Query {
  allOperations: [Operation]
  operation(id: ID!): Operation
  trace(id: ID!): Trace
  allTypes: [Type]
}

type Type {
  key: ID
  parentType: String
  fieldName: String
  returnType: String
  startOffset: Int
}
```

```typescript
// api/src/resolvers.ts

// ...
  Query: {
    allOperations: () => Operations.allOperations(),
    operation: (_, { id }) => Operations.get(id),
    trace: (_, { id }) => Traces.get(id),
    allTypes: () => Types.allTypes(), // <- newly added
  },
```

We'll have to call the init script from the index as well to boot the view:

```typescript
// ...
import { Operations, Traces, Resolvers } from './models';
// ...
app.listen(8000, () => {
  console.log('API started http://localhost:8000/graphiql');
  db
    .connect()
    .then(() => db.query(`create extension if not exists "uuid-ossp";`))
    .then(() => Operations.init())
    .then(() => Traces.init())
    .then(() => Resolvers.init()) // <- newly added
    .catch(err => {
      console.error('pg error:', err);
      process.exit(1);
    });
});
```

# Adding simple routing to UI

Now if we go back to the UI, we'll realize that we could do simple component switch but I want to render components based off the URL. So I like to use [this method](https://hackernoon.com/routing-in-react-the-uncomplicated-way-b2c5ffaee997) for routing. We'll do ours a little differently. If you prefer using `react-router` instead of `history` you can scroll down to the **Adding a Schema table** section.

Start by installing [history](https://www.npmjs.com/package/history) with `yarn add history`, the create a new file `ui/src/routes.js`:

```jsx
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

```jsx
// ui/src/pages/NotFound.js
import React from 'react';

const NotFound = () => <p>404 - Not Found</p>;

export default NotFound;
```

Now let's create a `history` module to handle browswer history, routing and change detection.

```jsx
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

```jsx
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

```jsx
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

# Adding a table for schema types

We're now ready to keep going with our dashboard and make a `Schema` component to display the list of `allTypes` we created earlier.

```jsx
// ui/src/components/schema/Schema.js
import './Schema.css';

import gql from 'graphql-tag';
import React from 'react';
import { Query } from 'react-apollo';

import { groupBy } from '../../utils';

const allTypes = gql`
  {
    allTypes {
      key
      parentType
      fieldName
      returnType
    }
  }
`;

const Schema = () => (
  <table className="Schema">
    <thead>
      <tr>
        <th>Type</th>
        <th>Field</th>
        <th>Return Type</th>
      </tr>
    </thead>
    <SchemaTypes />
  </table>
);

const SchemaTypes = () => (
  <Query query={allTypes}>
    {({ loading, error, data }) => {
      if (loading) return RowSpan('Loading...');
      if (error) return RowSpan(error.message);

      const types = groupBy(data.allTypes, t => t.parentType);
      return Object.entries(types).map(([parentType, fields], i) => {
        return (
          <tbody key={i}>
            <tr>
              <td>{parentType}</td>
            </tr>
            {fields.map(FieldRow)}
          </tbody>
        );
      });
    }}
  </Query>
);

const FieldRow = ({ key, fieldName, returnType }) => (
  <tr key={key}>
    <td />
    <td>{fieldName}</td>
    <td>{returnType}</td>
  </tr>
);

const RowSpan = content => (
  <tbody>
    <tr>
      <td colSpan="3">{content}</td>
    </tr>
  </tbody>
);

export default Schema;
```

Awesome work everyone! If we checkout the page (hopefully we've been checking on it now and again for issues) We should now see something like this:

![Schema table view]({{ "/assets/img/schema-types.png" | absolute_url }})

# Metrics

Now for the juicy stuff! I promised you usage statistics in the intro so here we are finally! Let's calculate the usage percentage of a field inside a type. We will need the count of queries made with each type and the count of queries containing each field. We'll have to add on to the `allTypes` query like so:

```typescript
// api/src/models/types.ts
import { db } from '../connectors';

export class Types {
  static allTypes() {
    return db
      .query(
        `with p as (
          select "parentType",
                  count(*) as parent_count
          from resolver
          group by 1
        ), f as (
          select "parentType",
                 "fieldName",
                 count(*) as field_count
          from resolver
          group by 1, 2
        )
        select md5(row(f."parentType", f."fieldName")::text) as key,
               f."parentType",
               f."fieldName",
               round((field_count * 100)::numeric / parent_count, 1) as usage
        from p
        join f on p."parentType" = f."parentType";`
      )
      .then(res => res.rows);
  }
}
```

Let's deconstruct this query again to explain what is happening:

```sql
select "parentType",
        count(*) as parent_count
from resolver
group by 1;

 parentType | parent_count
------------+--------------
 User       |          226
 Query      |          166
```

The `with as` syntax allows us to create a temporary table essentially to make it easier to build a complex query. So we see here we have the total number of queries with all the different types.

```sql
 select "parentType",
        "fieldName",
        count(*) as field_count
from resolver
group by 1, 2;

 parentType | fieldName | field_count
------------+-----------+-------------
 User       | name      |          73
 User       | parent    |           2
 User       | codes     |          34
 User       | email     |          51
 User       | age       |          66
 Query      | fn        |          58
 Query      | me        |         108
```

The second `with` query gets us the total number of queries with a given type and field! Now the third `with` query is there for a lack of a better understanding. I'm sure there is a more efficient way to get the `returnType` but I couldn't crack it. Perhaps a `SQL` wizard reading this can help me out. Now all we need to do is join these queries and add some math to get the percentage!

```sql
with p as (
  select "parentType",
          count(*) as parent_count
  from resolver
  group by 1
), f as (
  select "parentType",
          "fieldName",
          count(*) as field_count
  from resolver
  group by 1, 2
), r as (
  select "parentType",
          "fieldName",
          "returnType"
  from resolver
  group by 1, 2, 3
)
select md5(row(f."parentType", f."fieldName")::text) as key,
        f."parentType",
        f."fieldName",
        r."returnType",
        round((field_count * 100)::numeric / parent_count, 1) as usage
from p
join f on p."parentType" = f."parentType"
join r on f."parentType" = r."parentType" and f."fieldName" = r."fieldName";

               key                | parentType | fieldName | returnType | usage
----------------------------------+------------+-----------+------------+-------
 36c67cfe4fd186c2fa98ca431069cb56 | User       | name      | String     |  32.3
 888a5fba1f296404df873c2f0549c7b5 | User       | parent    | User       |   0.9
 ebc6ce5cdc9722dbe998ec219eeae919 | User       | codes     | [Int]      |  15.0
 95331ca259e940d1692a20b50b1fe746 | User       | email     | String     |  22.6
 ff428b6b6708d3fcdbb453d6117be2a3 | User       | age       | Int        |  29.2
 7a8bf89734f62dee77063596fe8caac1 | Query      | fn        | User       |  34.9
 09cc45d11bdd6877bfddb0465fcb3d12 | Query      | me        | User       |  65.1
```

Super cool! Now we add `usage` to the `type Type`:

```graphql
# api/src/schema.graphql

# ...
type Type {
  key: ID
  parentType: String
  fieldName: String
  returnType: String
  usage: Float
}
# ...
```

Nice! Now if we go back to the `ui` `Schema.js`:

```jsx
// ...
const allTypes = gql`
  {
    allTypes {
      key
      parentType
      fieldName
      returnType
      usage
    }
  }
`;

const Schema = () => (
  <table className="Schema">
    <thead>
      <tr>
        <th>Type</th>
        <th>Field</th>
        <th>Return Type</th>
        <th>Usage %</th>
      </tr>
    </thead>
    <SchemaTypes />
  </table>
);

// ...

const FieldRow = ({ key, fieldName, returnType, usage }) => (
  <tr key={key}>
    <td />
    <td>{fieldName}</td>
    <td>{returnType}</td>
    <td>{usage}%</td>
  </tr>
);

// ...
```

![Schema table view with usage percentage]({{ "/assets/img/usage.png" | absolute_url }})

Awesome! We've got very useful metric now that can help us make better API design decisions based on usage patterns of clients or look for optimization pathways to faster queries. This was a longer post than usual so thanks for reading! Stay tuned for more!

## Repo links

* [api](https://github.com/mpicard/graphql-metrics-api/tree/part-5)
* [ui](https://github.com/mpicard/graphql-metrics-ui/tree/part-5)
