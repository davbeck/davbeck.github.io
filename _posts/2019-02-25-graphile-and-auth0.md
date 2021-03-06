---
layout: post
title: Rapid API Development with PostGraphile and Auth0
date: 2019-02-25
tags: [PostGraphile, GraphQL, PostgreSQL, Auth0]
---

[PostGraphile](https://www.graphile.org/postgraphile/) is a really great tool for [creating api backends very quickly primarily using PostgreSQL](http://localhost:4000/posts/2019-02-12-rapid-api-development-with-postgraphile). However, it does require you to rethink how you make apis. This is especially true for authorization and authentication. But it certainly is possible (even easy) to create secure apis using Graphile.

## An example

Let's look at what it takes to make an api for a web app that includes login and security. [TodoMVC](http://todomvc.com) is a project that tries to impliment the same web app in multiple different frameworks. I'm using an [implimentation by chriswiles](https://github.com/ChrisWiles/React-todoMVC) as a starting point. It uses a more recent version of React than the official React implimentation.

If you run the project locally (or try it out at [chriswiles.github.io/React-todoMVC](https://chriswiles.github.io/React-todoMVC/)) you can add todo items, remove them, mark them complete, filter, etc. But, alas, if you refresh they will all be gone (that's not the right way to clear your todos).

Let's fix that by adding an api and saving results to the cloud ☁️🌈.

I've created a basic GraphQL api using PostGraphile and integrated that into the client using Apollo. You can see that on the [`no-auth`](https://github.com/davbeck/Postgraphile-todoMVC/tree/no-auth) branch in [the sample project](https://github.com/davbeck/Postgraphile-todoMVC).

I'm using [dbmate](https://github.com/amacneil/dbmate) to handle the migrations. You can see the entire schema under `db/schema.sql`. The relavant parts for the api are the table and 2 functions:

```sql
CREATE TABLE app.todo (
    id integer NOT NULL,
    title text,
    completed boolean DEFAULT false NOT NULL,
    "order" integer DEFAULT 0 NOT NULL,
    created_at timestamp with time zone DEFAULT now() NOT NULL,
    updated_at timestamp with time zone DEFAULT now() NOT NULL
);

CREATE FUNCTION app.clear_completed() RETURNS SETOF app.todo
    LANGUAGE sql
    AS $$
DELETE FROM app.todo WHERE completed = true RETURNING *;
$$;

CREATE FUNCTION app.complete_all() RETURNS SETOF app.todo
    LANGUAGE sql
    AS $$
UPDATE app.todo
   SET completed = NOT (SELECT COUNT(*) FROM app.todo WHERE completed = true) = (SELECT COUNT(*) FROM app.todo)
RETURNING *;
$$;
```

It's kind of cool to see how little it takes to create a basic api using Graphile. [Todo-Backend](https://www.todobackend.com) is a similar project that impliments an api for todo items in multiple languages and frameworks. That project assumes a REST api, so our backend won't be compatable with that, but at this point we have everything defined in that spec.

## Adding authentication

This is the part where I'm suppose to say, "this is an improvement but we can make it better...". But, it's not *really* an improvement if you think about it. Now when you refresh, or open the site on a different device, you're changes are still available, but they are also available for everyone! That's not very good for privacy.

What we need is to add a login and only show the current user's todo items. We need an api that not only filters what the user wants to see, but also enforces any malicious users from accessing someone else's items.

You can create your own login system using Graphile. [The docs have a pretty good tutorial on that](https://www.graphile.org/postgraphile/postgresql-schema-design/). But, for rapid development, you'll want something pre-built. That is where [Auth0](https://auth0.com) comes in. It provides the basics for creating accounts and login, but also things like forgot password, single sign on with sites like Google and Facebook, 2 factor authentication, and much much more.

<span class="side-note">
If you want to follow along with the sample code and run it yourself, you'll need to [create an Auth0 account](https://auth0.com/signup) and add your account values to the `.env` files.
</span>

There are a lot of ways to authenticate with Auth0, especially depending on what platform (web, iOS, Android) you are using. For our React app, I mostly used the [quickstart guide from Auth0](https://auth0.com/docs/quickstart/spa/react/01-login) with 2 modifications. First, instead of showing a login button when a new user opens the page, I immediately redirect to the Auth0 login page. User's won't be able to access anything until they login, so why not skip a step for them. Second, when I get an access token I update a token cookie with it's value. This will automatically get passed to our api (see [the apollo documentation for more info](https://www.apollographql.com/docs/react/recipes/authentication.html#Cookie)). You can see this in [`client/src/Auth.js`](https://github.com/davbeck/Postgraphile-todoMVC/blob/authentication/client/src/Auth.js).

Next, we need to parse that token in our api and pass it into Graphile and Postgres. Graphile has a config method called [`pgSettings`](https://www.graphile.org/postgraphile/usage-library/#pgsettings-function) that allows you to pass in values to the Postgres transaction, including a role:

```js
pgSettings: async req => {
  try {
    const claimes = await parseClaims(req);

    return {
      role: "todo_user",
      "user.id": claimes.sub,
    };
  } catch (error) {
    console.error("failed to authenticate", error);
    return { role: "todo_anonymous" };
  }
},
```

You can see how the token gets parsed in [auth.js](https://github.com/davbeck/Postgraphile-todoMVC/blob/master/auth.js). We are also setting a default role (`todo_anonymous`) if the user is not authenticated. There are cases where an unauthenticated user can still use parts of your api, for instance on Twitter anyone can see public profiles. That's not true here, but it's still important to set a limited role as the default to keep unauthenticated users from having full access.

But, we don't have a `todo_user` or `todo_anonymous` role yet. Let's create that along with in a migration:

```sql
CREATE ROLE todo_anonymous;
CREATE ROLE todo_user;

GRANT USAGE ON SCHEMA app TO todo_anonymous, todo_user;

GRANT SELECT ON app.todo TO todo_user;
GRANT INSERT (title, completed, "order") ON app.todo TO todo_user;
GRANT UPDATE (title, completed, "order") ON app.todo TO todo_user;
GRANT DELETE ON app.todo TO todo_user;
GRANT USAGE ON SEQUENCE app.todo_id_seq to todo_user;
```

<span class="side-note">
While most things in Postgres are constrained to the database they are created in (including schemas, tables and functions) roles are global to a Postgres server isntance, so it's important that we prefix them. If we just called this role something like "user" it would clash with other projects that also used that role name on the same server. This is especially common when working locally. Also, roles don't show up in schema.sql generated by dbmate.
</span>

Here we create our roles and give `todo_user` access to our todo table (including the auto increment id function which is required for INSERT). We don't want clients manually setting an id, update_at, so we only grant permissions for updating and inserting specific fields.

If you run those migrations and try the app out at this point (see the [`authentication`](https://github.com/davbeck/Postgraphile-todoMVC/tree/authentication) branch in the sample code), you should be redirected to login, and then have access to the api just like before. If you try opening graphiql and sending requests without logging in, you'll get access errors, meaning the api is secure from unauthenticated users.

## Adding authorization

However, we still have our original problem, which is that everyone is sharing a single todo list. Anyone can create an account, and from there anyone has access to everyone else's todo items. This is the difference between authentication and authorization.

In order to authorize access to specific resources, we will use [row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html). But first, we need to mark todo items with their user:

```sql
CREATE SCHEMA app_hidden;
GRANT USAGE ON SCHEMA app_hidden TO todo_anonymous, todo_user;

CREATE OR REPLACE FUNCTION app_hidden.current_user_id() RETURNS text AS $$
  SELECT nullif(current_setting('user.id', true), '')::text;
$$ LANGUAGE sql STABLE;
GRANT EXECUTE ON FUNCTION app_hidden.current_user_id() TO todo_anonymous, todo_user;

DELETE FROM "app"."todo";
ALTER TABLE "app"."todo" ADD COLUMN "user_id" text NOT NULL DEFAULT app_hidden.current_user_id();
```

We're creating our 3rd schema here (`app_hidden`). This lives between `app` and `app_private`. This schema will not be exposed to the api, but will be usable from other functions, or in this case, as a default value. That's different from `app_private` which is inaccessible for security reasons.

Graphile sets any values we return from `pgSettings` in Postgres, so we can access it using `current_setting`. Here we add a function to access that setting more easily.

Finally, we add a `user_id` column to our todo table, using the current user id as a default value. We have to delete the existing items though because they don't have a user associated with them and Postgres would complain about our `NOT NULL` constraint.

If you create some new todo items now and view them in the database, you can see that they have a `user_id` set.

Next let's use role level security to make sure users can only see their own items:

```sql
ALTER TABLE app.todo ENABLE ROW LEVEL SECURITY;

CREATE POLICY select_todo ON app.todo FOR SELECT TO todo_user
 USING (user_id = app_hidden.current_user_id());
```

What's really cool here is that not only does it enforce who can see what rows from a security standpoint, but Postgres will also automatically filter out any rows that don't match our condition. Try creating a few results using different user_ids and the api will only return the ones matching the current user.

<span class="side-note">
The condition on policies can be quit complex too. For instance, in a project I'm working on, I control access to a "team" table based on a separate "membership" table as well as an `is_admin` column on the "user" table.
</span>

Let's do the same for insert, update and delete:

```sql
CREATE POLICY insert_todo ON app.todo FOR INSERT TO todo_user
  WITH CHECK (user_id = app_hidden.current_user_id());

CREATE POLICY update_todo ON app.todo FOR UPDATE TO todo_user
 USING (user_id = app_hidden.current_user_id());

CREATE POLICY delete_todo ON app.todo FOR DELETE TO todo_user
 USING (user_id = app_hidden.current_user_id());
```

<span class="side-note">
Alternatively, if all your policies use the same restrictions like they do here, you can set all 4 CRUD policies with `CREATE POLICY all_own ON app.todo FOR ALL TO todo_user USING (user_id = app_hidden.current_user_id());`. Other times you might have different rules for different operations. A blog for instance would allow anyone to `SELECT` but only the author or admin to update.
</span>

With that, the api is secure. You have to have an account to access todo items, you can only see your own todo items, you can only create todos that are assigned to you (and that gets set automatically), you can only update your own items (and you can't change a user_id to or from someone else's), and you can only delete your own items.

## Cleaning up the api

If you poke around GraphiQL (or the GraphQL schema) you'll notice that `createTodo` and `updateTodo` include fields that the user can't actually edit:

<img src="/images/2019-02-25-graphile-and-auth0/api-cleanpup.png" />

If a client actually tries to set these fields, Postgres will block the change based on our grants above. However it's fairly messy to have these hanging around the schema.

You could remove the fields using [smart comments](https://www.graphile.org/postgraphile/smart-comments/), but a cleaner approach is to use have Graphile detect what can and cannot be edited.

Graphile has an option `ignoreRBAC` (ignore role-based access control) which is currently true by default. When set to false it will remove any operations or fields that the connection doesn't have access to. There is 1 catch though: so far we've been using an admin user to connect to Postgres with. We control access using roles, but the the schema is generated using the user we connect to the database with. In order for RBAC to take effect in the schema we need to create a new role/login for Graphile.

In a Postgres console:

```sql
CREATE ROLE todo_graphile LOGIN NOINHERIT PASSWORD 'password';
```

Or in a terminal shell:

```sh
psql -c "CREATE ROLE todo_graphile LOGIN NOINHERIT PASSWORD 'password';"
```

We don't want to put this in a migration because on a production server, we would want to give this role a very long and secure password.

Next, we give this new user access to each of our other roles.

```sql
GRANT todo_user TO todo_graphile;
GRANT todo_anonymous TO todo_graphile;
```

Then the connetion string used with Graphile needs to be changed to use that new user. In the sample project, I've added `GRAPHILE_URL` to the .env file and switched to using that variable in index.js. The original `DATABASE_URL` is still needed to run migrations using dbmate.

Now the schema will include only items that one of it's child roles include (in this case, just what todo_user has access to). If you have multiple roles for users (for instance, a regular user role and an admin role) the schema will include whatever the most privileged user has access to, but Postgres will still enforce restrictions if a less priviledged tries to make a request they aren't allowed to make.

---

If you are familiar with securing apis using manual checks from a traditional backend, enforcing your api with upfront declarations like this can be quit uncomfortable. But when you embrace it, I think you'll find it's actually quit secure and simple.

<span class="side-note">
**UPDATE:** Thanks to Graphile maintainer [Benjie](https://twitter.com/Benjie) for reaching out and pointing out some typos as well as suggestions on using RBAC instead of smart comments.
</span>