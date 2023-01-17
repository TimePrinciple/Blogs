# Workflow of `lfs-test-server`

## main

> tcpKeepAliveListener sets TCP kkep-alive timeouts on accepted connections. It's used by ListenAnd ServeTLS so dead TCP connections (e.g. closing laptop mid-download) eventually go away.

1. Check for flag `-v` in input arguments, print *version* information if found.
2. Construct a `Listener`.
3. Destruct into a *Tracking Listener* (tl) and an *err* (if happened) returned by function `NewTrackingListener`. Log the message if an error is found.
4. Check for connection type (HTTP or HTTPS) in config, apply function `wrapHttps` if flag `IsHTTPS` in config is `true`.
5. Destruct into a *meta store* and au *err* (if happened) returned by function `NewMetaStore`. Log the message if an error is found.
6. Destruct into a `contentStore` and an *err* (if happened) returned by function `NewContentStore`. Log the message if an error is found.
7. ?Check for status of each Listener before closing the connections.
8. Start *TusServer* and store uploads (uploaded content?) in the given contentPath.
9. Create app returned by `NewApp` with `contentStore` and `metaStore`.
10. Start Tus Server.
11. Server the Listener (previous tl).
12. Wait until the process is completed.
13. Stop Tus Server.


### `NewTrackingListener`

- Tracking Listener tracks incoming connections so that application shutdown can wait until all in progress connections are finished before exiting.
- `NewTrackingListener` creates a new Tracking Listener, listening on the supplied address.
- Arguments: `string addr`.
- Returns: A pointer (or reference) to the created Tracking Listener, and an err if exist.

### `wrapHttps`

- Set `NextProtos` and `Certificates` by call to `tls.LoadX509KeyPair`.
- Construct a `tlsListener` from `TCPListener` and config.
- Arguments: `TrackingListener tl`, `config::cert`, `config::key`
- Returns: `tlsListener`

### `NewMetaStore`

- MetaStore implements a metadata storage. It stores user credentials and Meta information for objects. The storage is handled by boltdb.
- NewMetaStore creates a new MetaStore using the boltdb database at dbFile.
- Establish connection with database.
- Update database with received information.
- Arguments: `string dbFile`
- Returns: A pointer (or reference) to the `MetaStore` database.

### `NewContentStore`

- ContentStore provides a simple file system based storage.
- NewContentStore creates a ContentStore at the base directory.
- Arguments: `string base`
- Returns: A reference to `ContentStore`

### `NewApp`

- NewApp creates a new App using the ContentStore and MetaStore provided.
- Use `Router` to communicate with client for information.
- Arguments: `ContentStore content`, `MetaStore meta`
- Returns: constructed `app`.

