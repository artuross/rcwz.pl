---
title: A case against migration files
date: 2025-04-03T18:50:00+02:00
---

I never really understood the popularity of migration files. Sure enough, you need to somehow change schema from version A to version B and using SQL statements is the most straightforward way to do it, but surely, there has to be a better way, right..?

What if I told you... you don’t actually need migration files to manage your database schema — and your life would be way easier without them? No, I'm not advocating for an ORM.

## The problem

### What's the current schema?

The obvious issue with migration files is that you simply can't tell the final schema without inspecting the database. Imagine if you changed your backend service code with `ALTER` statements; Github diffs would certainly be more... entertaining.

But let's be honest — you're probably using some GUI tool to poke around the database during development anyway, so I'll give it a pass.

### The `ALTER` statement

Every schema migration tool I’ve used required to add at least one new file for the `up` migration. Some tools use the same file for `up` and `down` migration, some require one file for each.

But here’s the kicker: I already have TablePlus open during development, I’m probably going to use it to make schema changes instead of writing out `ALTER` statements by hand. I’ll likely be tweaking things as I go anyway.

So in the end, I still have to figure out the `up` part — and ideally the `down` one too.

- _Bruh? Just use AI._

Heh, what could possibly go wrong. &lt;insert DiCaprio laughing meme&gt;

### Uno reverse

Alright, you know this one. There's an `up` migration, but no `down` migration. After all, there's a reason why the windshield is bigger than the rearview mirror. But sometimes... you just gotta revert the change. Maybe it's not the schema — it's some other bug in the code.

I’ve seen projects with no `down` migrations at all. I've seen migrations that were adding and deleting things at the same time. Finally, I've seen migrations that not only were changing schema, they were migrating data too.

Why kill one bird when you can get 'em both?

### Everything else

There's more. For example, at one of my former companies, the _legacy_ (wink wink) monolith had probably hundreds of migration files. And because you can’t really get rid of them, they just pile up.

Another thing: most devs aren’t exactly database experts (I'm not). They don’t always understand the implications of their changes. Like — did you know that adding an index can block the entire DB?

Also, how is it normal to create a single PR that includes both a schema migration and code changes? There’s no way to deploy those changes exactly at the same time.

## The solution

The solution is so simple, so obvious... too obvious, in fact. Just... don't write migration files. Declare your schema declaratively the way you write Terraform configuration or Kubernetes manifests. A single `schema.sql` file that creates the entire schema from scratch will carry you a long way before you ever have to think about a word "migration".

While it’s technically still imperative SQL (`CREATE TABLE`, etc.), the experience feels declarative: you describe the desired end state, and the tooling figures out the steps to get there.

I actually first got to see it when I played with [PlanetScale](https://planetscale.com/docs/concepts/deploy-requests) (rip, the free tier) a few years back. Their UI and dev experience was something else. Then I found [`pg-schema-diff`](https://github.com/stripe/pg-schema-diff) made by Stripe and it's a game changer. I never want to see a migration file ever again. Please.

`pg-schema-diff` is pretty sweet. You just point it to a schema file and a live DB with previous schema and it will print all statements to migrate the schema. It can also do the migration on its own, but I think it's better to extract that part and report the statements in a PR for review.

This approach has some limitations. For example, you can't rename tables (it will do `CREATE TABLE` and `DROP TABLE`). Another thing is inability to do data migrations (but IMO it's a feature, most of the time).

I'm sure true SQL ninjas could point many more, valid cases, where preparing the statements yourself will be a better, faster and safer option. But at that point, the project probably has a dedicated database guru.

So if you have this starting schema:

```sql
CREATE TABLE teams (
    id              SERIAL    PRIMARY KEY,
    name            TEXT      NOT NULL
);

CREATE TABLE users (
    id              SERIAL    PRIMARY KEY,
    team_id         INTEGER   NOT NULL,
    email           TEXT      NOT NULL
);

CREATE INDEX users_email_idx ON users(email);
```

and you eventually figure out that current schema must be improved to prevent duplicates:

```diff
CREATE TABLE teams (
    id              SERIAL    PRIMARY KEY,
-   name            TEXT      NOT NULL
+   name            TEXT      UNIQUE NOT NULL
);

CREATE TABLE users (
    id              SERIAL    PRIMARY KEY,
    team_id         INTEGER   NOT NULL,
-   email           TEXT      NOT NULL
+   email           TEXT      UNIQUE NOT NULL
);

CREATE INDEX users_email_idx ON users(email);
```

You can run `pg-schema-diff plan` and will get a list of steps with explanation:

```txt
################################ Generated plan ################################
1. CREATE UNIQUE INDEX CONCURRENTLY teams_name_key ON public.teams USING btree (name);
	-- Statement Timeout: 20m0s
	-- Lock Timeout: 3s
	-- Hazard INDEX_BUILD: This might affect database performance. Concurrent index builds require a non-trivial amount of CPU, potentially affecting database performance. They also can take a while but do not lock out writes.

2. ALTER TABLE "public"."teams" ADD CONSTRAINT "teams_name_key" UNIQUE USING INDEX "teams_name_key";
	-- Statement Timeout: 3s

3. CREATE UNIQUE INDEX CONCURRENTLY users_email_key ON public.users USING btree (email);
	-- Statement Timeout: 20m0s
	-- Lock Timeout: 3s
	-- Hazard INDEX_BUILD: This might affect database performance. Concurrent index builds require a non-trivial amount of CPU, potentially affecting database performance. They also can take a while but do not lock out writes.

4. ALTER TABLE "public"."users" ADD CONSTRAINT "users_email_key" UNIQUE USING INDEX "users_email_key";
	-- Statement Timeout: 3s
```

How cool is that?

Now, there are still valid reasons to write migration files by hand, but for most projects that I have worked on, that’s overkill — it adds friction and increases mental overhead for no reason.
