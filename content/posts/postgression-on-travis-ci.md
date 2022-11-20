---
title: Using postgression on Travis CI
date: "2013-05-12"
aliases:
- /2013/05/12/postgression-on-travis-ci.html
---

At [Heroku](https://www.heroku.com) we use [Travis CI](http://about.travis-ci.org/docs/user/travis-pro/) to run project tests on push to [GitHub](https://github.com/). While Travis CI offers [PostgreSQL](http://about.travis-ci.org/docs/user/database-setup/#PostgreSQL) in their environment, it's version 9.1. A project I'm working on recently started using PostgreSQL 9.2's [JSON data type](http://www.postgresql.org/docs/9.2/static/datatype-json.html), which 9.1 does not have.

Needing 9.2, I searched for ways to make it available in the Travis CI environment. I found guides that suggested [upgrading PostgreSQL](http://reefpoints.dockyard.com/ruby/2013/03/29/running-postgresql-9.2-on-travis-ci.html) in a `before_script` but I didn't have much luck with that approach. Plus, it would add time to each build which I was hoping to avoid. I knew [Heroku Postgres](https://postgres.heroku.com/) offered 9.2 [by default](https://postgres.heroku.com/blog/past/2013/4/18/postgres_92_now_default/), what I really wanted was a fresh Heroku Postgres database for every test run.

Then I remembered [postgression](http://www.postgression.com/).

postgression is a service from [rdegges](http://www.rdegges.com/) and [zaidos](http://zaidox.com/), built atop Heroku Postgres. It has a very simple API:

```bash
$ curl http://api.postgression.com
postgres://user:password@host/db
```

The returned URL points to a Heroku Postgres database (running 9.2) which is automatically destroyed in 30 minutes, making it perfect for a Travis CI run.

My project connects to the database specified by [`$DATABASE_URL`](http://12factor.net/config), so I needed to get that set to the URL returned by postgression. Before I started using the JSON data type, I used the [`env`](http://about.travis-ci.org/docs/user/build-configuration/#Set-environment-variables) section of `.travis.yml` to set `$DATABASE_URL` to `postgres://localhost/ci` and created the `ci` database in the `before_script` section. This worked well as settings in the `env` section are made available to all parts of the build: `before_script`, `script`, etc. To use a postgression database for `$DATABASE_URL`, I wanted the equivalent of:

```bash
export DATABASE_URL=$(curl http://api.postgression.com)
```

The `env` section documentation has a note about needing to quote settings with asterisks (`*`) which got me wondering if specified settings were being evaluated by a shell. As an experiment, I changed my `env` section to:

```yaml
env:
- DATABASE_URL=$(curl http://api.postgression.com)
```

and it worked! The build output also confirmed this happened just once for the build instead of, say, once for `before_script` and again for `script`. This meant I had access to the same database in all parts of the build.

For a complete example in the wild, check out the [`.travis.yml`](https://github.com/erlware/dikdik/blob/e318019ca7e6c31bf50de15d2169382dc6a6f53f/.travis.yml) for [dikdik](https://github.com/erlware/dikdik), an erlang hstore interface. Since dikdik also requires PostgreSQL 9.2, [tsloughter](https://github.com/tsloughter) adopted the approach described here to test it with Travis CI.

Thanks to a union of these two great services, testing projects requiring PostgreSQL 9.2 is easy. Please support them if you find them valuable!
