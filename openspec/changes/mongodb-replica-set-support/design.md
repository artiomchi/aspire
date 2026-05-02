## Context

Aspire's MongoDB hosting (`Aspire.Hosting.MongoDB`) starts a standalone MongoDB container. The health check uses `AspNetCore.HealthChecks.MongoDb` for basic connectivity. The connection string is built by `MongoDBServerResource.BuildConnectionString()` using `ReferenceExpressionBuilder` and always includes authentication credentials (Aspire always generates a password).

MongoDB requires a keyfile for intra-cluster authentication when running `--replSet` with auth enabled. For a single-node replica set the keyfile content is never actually used for peer authentication (there are no peers), but MongoDB still requires the file to exist with permissions no more permissive than 0600 and readable by the `mongodb` process user (UID 999 in the official image).

The replica set must be explicitly initiated via `rs.initiate()` after `mongod` starts. This cannot be done in `docker-entrypoint-initdb.d` scripts because those run against a temporary `mongod` instance that does not use the `--replSet` flag.

## Goals / Non-Goals

**Goals:**
- Allow opt-in single-node replica set mode for local development and testing
- Gate dependent resource startup on the replica set having an elected primary
- Produce connection strings that work correctly for both host-side and container-to-container clients
- Be safe for persistent containers (stable across AppHost restarts)

**Non-Goals:**
- Multi-node replica sets
- Production replica set configuration
- Replica set support in the polyglot (ATS) app host path
- Changes to `Aspire.MongoDB.Driver` (the client component)

## Decisions

### D1: Keyfile injected via `WithContainerFiles`, content deterministic

**Decision:** Generate the keyfile content as `Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes($"{resourceName}-mongodb-keyfile")))` and inject it via `WithContainerFiles("/tmp", [...], defaultOwner: 999, defaultGroup: 999)`.

**Alternatives considered:**
- *Shell script in `docker-entrypoint-initdb.d`*: Scripts in that directory only run when the data directory is empty (first container creation). With `WithDataVolume()`, a persistent container would lose the keyfile from `/tmp` on restart since it is not in the data volume, and the init script would not re-run. This causes `mongod` to fail to start.
- *Random bytes per container creation*: `WithContainerFiles` for persistent containers uses file content as part of the lifecycle key — if content changes between AppHost runs, the persistent container is recreated, defeating persistence.
- *Bind-mounting a host-side file*: Requires writing to the host filesystem and complicates the developer experience.

Deterministic SHA256 of the resource name produces stable content across AppHost restarts (safe for persistent containers), is unique per resource, and avoids any shell/openssl dependency.

The `defaultOwner: 999` ensures the file is owned by the `mongodb` user inside the container. Mode `UserRead | UserWrite` (0600) satisfies MongoDB's keyfile permission check.

### D2: Replica set initiation in the health check, not `OnResourceReady`

**Decision:** Perform `rs.initiate()` inside the health check, and only return `Healthy` once `isWritablePrimary` is true.

**Alternatives considered:**
- *`OnResourceReady` callback*: Fires after the health check passes. This means the resource is already marked healthy before the replica set is initiated, creating a window where dependent services can connect to a non-writable node.
- *`docker-entrypoint-initdb.d` script calling `mongosh`*: Cannot initiate the replica set here because those scripts run against a standalone mongod; the `--replSet` flag is only present on the final mongod process.
- *Separate secondary health check*: Requires the primary health check to pass first, same ordering problem as `OnResourceReady`.

Placing initiation in the health check means the resource is only marked healthy after a primary is elected, which is exactly when dependent services should start. `rs.initiate()` is idempotent (error code 23 = `AlreadyInitialized`); the flag `_replicaSetInitiated` (thread-safe via `Interlocked.CompareExchange`) avoids redundant admin commands on subsequent health check invocations.

### D3: Custom `IHealthCheck` implementation, not `AddMongoDb` extension

**Decision:** Register a custom internal `IHealthCheck` class (`MongoDBServerHealthCheck`) that handles both the standalone and replica set cases, replacing the existing `AddMongoDb()` call.

**Rationale:** `AddMongoDb()` from `AspNetCore.HealthChecks.MongoDb` only performs connectivity checks. We need custom logic for replica set initiation and primary election polling, so a custom implementation is the simplest path. The existing behaviour for non-replica-set resources is preserved as the else-branch.

### D4: `directConnection=true` in connection string, not `replicaSet=<name>`

**Decision:** Append `directConnection=true` to the connection string when `ReplicaSetName` is set.

**Rationale:** `directConnection=true` tells the MongoDB driver to connect directly to the specified host without topology discovery. This is necessary because the replica set member host used in `rs.initiate()` is the internal container hostname (`resourceName:27017`), which differs from the host-accessible address. Without `directConnection=true`, the driver would attempt to resolve the internal member hostname from outside the container network and fail.

Adding `replicaSet=<name>` alongside `directConnection=true` is explicitly discouraged by MongoDB driver documentation as the behaviours conflict.

### D5: `rs.initiate()` member host uses the resource (container) name

**Decision:** Use `$"{mongoDBContainer.Name}:{DefaultContainerPort}"` as the member host in the `replSetInitiate` command.

**Alternatives considered:**
- *`127.0.0.1:27017`*: Works for the host-side health check client (which bypasses topology discovery via `directConnection=true`), but breaks container-to-container clients such as MongoExpress — from another Docker container `127.0.0.1` resolves to that container itself, not the MongoDB container.

Using the container name ensures the replica set member is resolvable from any container on the same Docker network. The host-side connection string's `directConnection=true` prevents the driver from trying to resolve this internal hostname from outside.

## Risks / Trade-offs

**[Risk] Health check performs write-like operation (`rs.initiate()`)**
→ The first health check invocation against a fresh replica set container sends an admin command that mutates server state. This is intentional and idempotent, but differs from the typical read-only health check pattern. The `_replicaSetInitiated` flag ensures initiation is attempted at most once per AppHost lifetime.

**[Risk] MongoDB UID 999 assumption**
→ The official `mongo` Docker image uses UID 999 for the `mongodb` user. A custom or future image with a different UID would cause the keyfile to be unreadable by `mongod`. This is documented and acceptable for a dev-only feature targeting the official image.

**[Risk] Persistent container restart with `/tmp` keyfile**
→ `/tmp` is not persistent across container restarts (stop/start without recreation). However, `WithContainerFiles` re-injects the keyfile on every container creation, not every start. If the container is restarted (not recreated), the keyfile in `/tmp` may be lost. Since the deterministic keyfile content is stable, DCP will not recreate the persistent container between runs, meaning restarts within a single AppHost session should retain `/tmp` contents. If `/tmp` is wiped on restart, `mongod` will fail. A more robust location (e.g., `/data/configdb/`) could be used if this proves to be a problem in practice.

**[Trade-off] `WithReplicaSet` not available in polyglot app hosts**
→ The `WithContainerFiles` overload used for keyfile injection is marked `[AspireExportIgnore]` and is not available to polyglot (ATS) app hosts. `WithReplicaSet` itself will also need `[AspireExportIgnore]` or a polyglot-compatible alternative. For the initial implementation, polyglot support is out of scope.

## Open Questions

- Should `WithReplicaSet` be `[AspireExportIgnore]` entirely, or is there a polyglot-friendly path worth considering now?
- Should the keyfile location be `/tmp/mongodb-keyfile` or a more persistent path like `/data/configdb/mongodb-keyfile` to handle container stop/start robustly?
