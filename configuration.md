# Configuration

Hockeypuck reads a TOML-format configuration file for setting various
options on the subsystems and features of the service.

## Logging

```
[hockeypuck]
logfile="/path/to/logfile"
loglevel=<one of: DEBUG,INFO,WARNING,ERROR,FATAL,PANIC>
```

If not configured, hockeypuck will log INFO level messages and higher severity
to standard error.

## Static HTML files

Hockeypuck will serve static files from `/` out of the `webroot` path, so long as
the path names do not conflict with HKP routed requests (like "/pks/lookup").

```
[hockeypuck]
webroot="/path/to/www/files"
```

`index.html` will be served by default if the path resolves to a directory and
the file exists.

## Custom HTML templates

By default, Hockeypuck will respond to HKP operations `op=index`, `op=vindex`
and `op=stats` requests with an `application/json` response. The underlying
structs for these responses can be used in HTML templates of your own design
to customize the output.

These options are:

```
[hockeypuck]
indexTemplate="/path/to/template"
vindexTemplate="/path/to/template"
statsTemplate="/path/to/template"
```

The path to the template must be a file containing a valid Go [html/template](https://golang.org/pkg/html/template/).

`indexTemplate` and `vindexTemplate` operate on a struct containing two top-level fields,

* .Query, an instance of the [hkp.Lookup](https://godoc.org/gopkg.in/hockeypuck/hkp.v1#Lookup) request parameters.
* .Keys, which is a slice of [jsonhkp.PrimaryKey](https://godoc.org/gopkg.in/hockeypuck/hkp.v1/jsonhkp#PrimaryKey) model structs.

`statsTemplate` operates on an instance of [server.stats](https://github.com/hockeypuck/server/blob/38c262ad65376d38727271cbbc5a71123672de70/server.go#L126).

See the [packaged template files](https://github.com/hockeypuck/packaging/tree/master/instroot/var/lib/hockeypuck/templates) examples.

## Storage

### MongoDB

If storage is not otherwise configured, the default is to use the MongoDB
driver to connect to `localhost:27017`. This is effectively:

```
[hockeypuck.openpgp.db]
driver="mongo"
dsn="localhost:27017"
```

The `dsn` field is just the _host:port_ of the MongoDB server.

Hockeypuck will default to database name `hkp` and collection name `keys` for storing public key material documents. This can be changed with the options:

```
[hockeypuck.openpgp.db.mongo]
db=dbname
collection=collection_name
```

### PostgreSQL

PostgreSQL >= 9.4 is required for use with Hockeypuck, as the JSONB data type is used to store most of the public key material. Some fields are broken out into separate columns for indexing. For details, refer to the PostgreSQL backend package, [pghkp.v1](https://gopkg.in/hockeypuck/pghkp.v1).

To use PostgreSQL:

```
[hockeypuck.openpgp.db]
driver="postgres-jsonb"
dsn="database=hkp host=/var/run/postgresql port=5433 sslmode=disable"
```

See the [pq driver package documentation](https://godoc.org/github.com/lib/pq) for details on how to construct the connection string.

## Peering

Hockeypuck supports the SKS reconciliation (recon) protocol. It can peer with
other Hockeypuck or SKS instances.

### Local peer options

```
[hockeypuck.conflux.recon]
httpAddr=":11371"
reconAddr=":11370"
```

The above are default settings if not otherwise specified.

`httpAddr` determines the address that will be advertised to remote peers for
retrieving key material with `/pks/hashquery` requests.

`reconAddr` determines the listen address for the recon server. This is
conventionally `:11370` among SKS keyservers.

### Adding remote peers

```
[hockeypuck.conflux.recon.partner.peer1]
httpAddr="keys.cmarstech.com:11371"
reconAddr="keys.cmarstech.com:11370"

[hockeypuck.conflux.recon.partner.peer2]
httpAddr="juju-azure-dev-y9157oo521.cloudapp.net:11371"
reconAddr="juju-azure-dev-y9157oo521.cloudapp.net:11370"
```

Create a section for each peer `[hockeypuck.conflux.recon.partner.peername]`,
where _peername_ is a unique logical name given to each peer (it doesn't have to relate
to the hostnames or anything).

For each peer, define the `httpAddr` and `reconAddr`. Note that these are
_usually_ the same host, but they might differ, especially if the peer
reverse-proxies their HKP service.

#### Protocol and network options

```
[hockeypuck.conflux.recon]
version="1.1.3"                # this is default
allowCIDRs=["10.0.0.1/8"]      # default is []
filters=["yminsky.dedup"]      # default is []
```

`version` is the protocol compatibility version (SKS release version)
advertised to remote peers. Hockeypuck does not use this field.

`allowCIDRS` is used to allow incoming recon connections from remote addresses
other than the defined peers. This is especially useful when inbound
connections to Hockeypuck are subject to NAT (some cloud providers do this). If
not specified, inbound connections are only allowed from partner IP addresses.

`filters` are labels that indicate the type of processing that has been applied
to key material. Recent versions of SKS typically require
`filters=["yminsky.dedup"]`, which indicates that duplicate PGP packets have
been dropped from key material. Hockeypuck deduplicates key material
regardless; this field is only used for protocol compatibility with SKS.

#### Prefix tree location

```
[hockeypuck.conflux.recon.leveldb]
path="/path/to/prefix/tree"
```

The prefix tree is a separate index maintained for synchronization purposes.
Hockeypuck uses LevelDB for its storage, but it is structurally similar to the
bdb prefix tree used by SKS.

The `path` given should be a writeable directory that already exists.
