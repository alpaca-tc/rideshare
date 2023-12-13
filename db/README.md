## Database Setup

## Goals

The *Principle of least privilege*[^prin] is followed by creating explicit `GRANT` commands for the `owner`, `app`, and `app_readonly` users.

The configuration is based on *My GOTO Postgres Configuration for Web Services*.[^gotocon] One of the other goals besides minimizing access, is to prevent accidental table drops.

Since the schema `rideshare` is created, the `public` schema is not needed and is removed.

For `psql` commands, use a `DATABASE_URL` environment variable that's set in your terminal.

The connection string connects to the Rideshare database, using the `owner` user. The value of `DATABASE_URL` is a connection string, with the format format `protocol://role:password@host:port/databasename`.

[^prin]: <https://en.wikipedia.org/wiki/Principle_of_least_privilege>
[^gotocon]: <https://tightlycoupled.io/my-goto-postgres-configuration-for-web-services/>

## Configuring Host Based Authentication (HBA)

You may want to configure *Host Based Authentication* (`HBA`)[^pghba].

Do that by editing your `pg_hba.conf` file. Changes in `pg_hba.conf` can be applied by *reloading* PostgreSQL.

## Reloading your PostgreSQL configuration

To reload your configuration, run: `pg_ctl reload` in your terminal. If you run into the following message, we'll get that addressed.

```sh
pg_ctl: no database directory specified and environment variable PGDATA unset
Try "pg_ctl --help" for more information.
```

This command assumes `PGDATA` is set and points to the data directory for your PostgreSQL installation.

Run `echo $PGDATA` to see the value. How do you set it if it's empty? Run the following commands in your terminal:

```sh
# Look at the value
psql -U postgres -c 'SHOW data_directory'

# Assign the value to PGDATA
export PGDATA="$(psql -U postgres \
  -c 'SHOW data_directory' \
  --tuples-only | sed 's/^[ \t]*//')"
echo "Set PGDATA: $PGDATA"
```

With `PGDATA` set, run `pg_ctl reload` again. Once PostgreSQL config reloads, you're all set.

[^pghba]: <https://www.postgresql.org/docs/current/auth-pg-hba-conf.html>

## Docker

Reset everything:

```sh
sh reset_docker_instances.sh
```

Tear down docker:

```sh
sh teardown_docker.sh
```

## Slow Clients

Replace `config/database.yml` (or just the "slow clients" section)

```
cp config/database-slow-clients.sample.yml config/database.yml
```

With that in place, create a model:

```ruby
class SlowClientModel < ApplicationRecord
  self.establish_connection :slow_clients
end
```

Run query code that takes 5 seconds, and verify that it's canceled in the normal configuration.

The "slow client" configuration allows it since it has a higher statement timeout configured.

```rb
Trip.connection.execute("SELECT PG_SLEEP(5)")
SlowClientModel.connection.execute("SELECT PG_SLEEP(5)").first
```

## pg_cron

[Scheduling maintenance with the PostgreSQL pg_cron extension](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/PostgreSQL_pg_cron.html)

- The extension is created using the postgres superuser
- The superuser grants usage privileges to the owner role, for the cron schema
- Now the owner user can schedule their own jobs, for objects they own

```sql
psql -U postgres -d rideshare_development;

CREATE EXTENSION pg_cron;

GRANT USAGE ON SCHEMA cron TO owner;
```

Run a job:
```sql
SELECT cron.schedule(
  'rideshare trips manual vacuum',
  '10 * * * *',
  'VACUUM (ANALYZE) rideshare.trips'
);
```

View the jobs:
```sql
SELECT * FROM cron.job;
```

View job runs:
```sql
SELECT * FROM cron.job_run_details;
```

![Screenshot of PgHero Scheduled Jobs](https://i.imgur.com/rxRf7Qn.png)

## active-record-doctor

Run the tool from your terminal:

```sh
bundle exec rake active_record_doctor:
```


## database_consistency

Run the tool from your terminal:

```sh
database_consistency
```


## postgresql-hll and active_hll

Instructions TBD
