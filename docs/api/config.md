# Config

## Overview

Mesop is configured at the application level using environment variables.

## Configuration values

### MESOP_STATIC_FOLDER

> **NOTE:** By default, this feature is not enabled, but in an upcoming release, the
default will be `static`.

Allows access to static files from the Mesop server.

It is important to know that the specified folder path is relative to the current
working directory where the Mesop command is run. Absolute paths are not allowed.

Example:

In this case, the current working directory is `/srv`, which means Mesop will make
`/srv/static` the static folder.

```bash
cd /srv
MESOP_STATIC_FOLDER=static mesop app/main.py
```

Here are some examples of valid paths. Let's assume the current working directory is
`/srv/`

- `static` becomes `/srv/static`
- `static/` becomes `/srv/static`
- `static/assets` becomes `/srv/static/assets`
- `./static` becomes `/srv/static`
- `./static/` becomes `/srv/static`
- `./static/assets` becomes `/srv/static/assets`

Invalid paths will raise `MesopDeveloperException`. Here are some examples:

- Absolute paths (e.g. `/absolute/path`)
- `.`
- `./`
- `..`
- `../`

### MESOP_STATIC_URL_PATH

This is the base URL path from which files for your specified static folder will be
made viewable.

The static URL path is only recognized if `MESOP_STATIC_FOLDER` is set.

For example, given `MESOP_STATIC_FOLDER=static` and `MESOP_STATIC_URL_PATH=/assets`, the
file `static/js/script.js` can be viewable from the URL path `/assets/js/script.js`.

**Default:** `/static`

### MESOP_STATE_SESSION_BACKEND

Sets the backend to use for caching state data server-side. This makes it so state does
not have to be sent to the server on every request, reducing bandwidth, especially if
you have large state objects.

The backend options available at the moment are `memory`, `file`, `sql`, and `firestore`.

#### memory

Users should be careful when using the `memory` backend. Each Mesop process has their
own RAM, which means cache misses will be common if each server has multiple processes
and there is no session affinity. In addition, the amount of RAM must be carefully
specified per instance in accordance with the expected user traffic and state size.

The safest option for using the `memory` backend is to use a single process with a
good amount of RAM. Python is not the most memory efficient, especially when saving data
structures such as dicts.

The drawback of being limited to a single process is that requests will take longer to
process since only one request can be handled at a time. This is especially problematic
if your application contains long running API calls.

If session affinity is available, you can scale up multiple instances, each running
single processes.

#### file

Users should be careful when using the `file` backend. Each Mesop instance has their
own disk, which can be shared among multiple processes. This means cache misses will be
common if there are multiple instances and no session affinity.

If session affinity is available, you can scale up multiple instances, each running
multiple Mesop processes. If no session affinity is available, then you can only
vertically scale a single instance.

The bottleneck with this backend is the disk read/write performance. The amount of disk
space must also be carefully specified per instance in accordance with the expected user
traffic and state size.

You will also need to specify a directory to write the state data using
`MESOP_STATE_SESSION_BACKEND_FILE_BASE_DIR`.

#### SQL

> NOTE: Setting up and configuring databases is out of scope of this document.

This option uses [SqlAlchemy](https://www.sqlalchemy.org/) to store Mesop state sessions
in supported SQL databases, such as SQLite3 and PostgreSQL. You can also connect to
hosted options, such as GCP CloudSQL.

If you use SQLite3, you cannot use an in-memory database. It has to be a file. This
option has similar pros/cons as the `file` backend. Mesop uses the default configuration
for SQLite3, so the performance will not be optimized for Mesop's usage patterns.
SQLite3 is OK for development purposes.

Using a database like PostgreSQL will allow for better scalability, both vertically and
horizontally, since the database is decoupled from the Mesop server.

The drawback here is that this requires knowledge of the database you're using. At
minimum, you will need to create a database and a database user with the right
privileges. You will also need to create the database table, which you can create
with this script. You will need to update the CONNECTION_URI and TABLE_NAME to match
your database and settings. Also the database user for this script will need privileges
to create tables on the target database.

```python
from sqlalchemy import (
  Column,
  DateTime,
  LargeBinary,
  MetaData,
  String,
  Table,
  create_engine,
)

CONNECTION_URI = "your-database-connection-uri"
# Update to "your-table-name" if you've overridden `MESOP_STATE_SESSION_BACKEND_SQL_TABLE`.
TABLE_NAME = "mesop_state_session"

db = create_engine(CONNECTION_URI)
metadata = MetaData()
table = Table(
  TABLE_NAME,
  metadata,
  Column("token", String(23), primary_key=True),
  Column("states", LargeBinary, nullable=False),
  Column("created_at", DateTime, nullable=False, index=True),
)

metadata.create_all(db)
```

The Mesop server will raise a `sqlalchemy.exc.ProgrammingError` if there is a
database configuration issue.

By default, Mesop will use the table name `mesop_state_session`, but this can be
overridden using `MESOP_STATE_SESSION_BACKEND_SQL_TABLE`.

#### GCP Firestore

This options uses [GCP Firestore](https://cloud.google.com/firestore?hl=en) to store
Mesop state sessions. The `(default)` database has a free tier that can be used for
for small demo applications with low traffic and moderate amounts of state data.

Since Firestore is decoupled from your Mesop server, it allows you to scale vertically
and horizontally without the considerations you'd need to make for the `memory` and
`file` backends.

In order to use Firestore, you will need a Google Cloud account with Firestore enabled.
Follow the instructions for [creating a Firestore in Native mode database](https://cloud.google.com/firestore/docs/create-database-server-client-library#create_a_in_native_mode_database).

Mesop is configured to use the `(default)` Firestore only. The GCP project is determined
using the Application Default Credentials (ADC) which is automatically configured for
you on GCP services, such as Cloud Run.

For local development, you can run this command:

```sh
gcloud auth application-default login
```

If you have multiple GCP projects, you may need to update the project associated
with the ADC:

```sh
GCP_PROJECT=gcp-project
gcloud config set project $GCP_PROJECT
gcloud auth application-default set-quota-project $GCP_PROJECT
```

Mesop leverages Firestore's [TTL policies](https://firebase.google.com/docs/firestore/ttl)
to delete stale state sessions. This needs to be set up using the following command,
otherwise old data will accumulate unnecessarily.

```sh
COLLECTION_NAME=collection_name
gcloud firestore fields ttls update expiresAt \
  --collection-group=$COLLECTION_NAME
```

By default, Mesop will use the collection name `mesop_state_sessions`, but this can be
overridden using `MESOP_STATE_SESSION_BACKEND_FIRESTORE_COLLECTION`.

**Default:** `none`

### MESOP_STATE_SESSION_BACKEND_FILE_BASE_DIR

This is only used when the `MESOP_STATE_SESSION_BACKEND` is set to `file`. This
parameter specifies where Mesop will read/write the session state. This means the
directory must be readable and writeable by the Mesop server processes.

### MESOP_STATE_SESSION_BACKEND_FIRESTORE_COLLECTION

This is only used when the `MESOP_STATE_SESSION_BACKEND` is set to `firestore`. This
parameter specifies which Firestore collection that Mesop will write state sessions to.

**Default:** `mesop_state_sessions`

### MESOP_STATE_SESSION_BACKEND_SQL_CONNECTION_URI

This is only used when the `MESOP_STATE_SESSION_BACKEND` is set to `sql`. This
parameter specifies the database connection string. See the [SqlAlchemy docs for more details](https://docs.sqlalchemy.org/en/20/core/engines.html#database-urls).

**Default:** `mesop_state_session`

### MESOP_STATE_SESSION_BACKEND_SQL_TABLE

This is only used when the `MESOP_STATE_SESSION_BACKEND` is set to `sql`. This
parameter specifies which SQL database table that Mesop will write state sessions to.

**Default:** `mesop_state_session`

### MESOP_PROD_UNREDACTED_ERRORS

Mesop, by default, only shows unredacted errors in debug mode.

If you run in prod mode, the errors are redacted (internal errors are given a generic message "Sorry, there was an error. Please contact the developer.") and the tracebacks are not shown.

If you want to show unredacted errors, including in prod mode, set this to `true`. This may be useful if you're deploying a Mesop app internally and want to get error details even in production.

## Experimental configuration values

These configuration values are experimental and are subject to breaking change, including removal in future releases.

### MESOP_CONCURRENT_UPDATES_ENABLED

!!! warning "Experimental feature"

      This is an experimental feature and is subject to breaking change. There are many bugs and edge cases to this feature.

Allows concurrent updates to state in the same session. If this is not updated, then updates are queued and processed sequentially.

By default, this is not enabled. You can enable this by setting it to `true`.

### MESOP_WEBSOCKETS_ENABLED

!!! warning "Experimental feature"

    This is an experimental feature and is subject to breaking change. Please follow [https://github.com/google/mesop/issues/1028](https://github.com/google/mesop/issues/1028) for updates.

This uses WebSockets instead of HTTP Server-Sent Events (SSE) as the transport protocol for UI updates. If you set this environment variable to `true`, then [`MESOP_CONCURRENT_UPDATES_ENABLED`](#MESOP_CONCURRENT_UPDATES_ENABLED) will automatically be enabled as well.

### MESOP_APP_BASE_PATH

This is the base path used to resolve other paths, particularly for serving static files. Must be an absolute path. This is rarely needed because the default of using the current working directory is usually sufficient.

## Usage Examples

### One-liner

You can specify the environment variables before the mesop command.

```sh
MESOP_STATE_SESSION_BACKEND=memory mesop main.py
```

### Use a .env file

Mesop also supports `.env` files. This is nice since you don't have to keep setting
the environment variables. In addition, the variables are only set when the application
is run.

```sh title=".env"
MESOP_STATE_SESSION_BACKEND=file
MESOP_STATE_SESSION_BACKEND_FILE_BASE_DIR=/tmp/mesop-sessions
```

When you run your Mesop app, the .env file will then be read.

```sh
mesop main.py
```
