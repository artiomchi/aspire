## Why

Applications that use MongoDB transactions or change streams require the server to be running as a replica set, but Aspire's MongoDB hosting only starts a standalone instance. Developers are currently unable to test these features locally through Aspire without manually configuring a replica set outside the AppHost.

## What Changes

- Add `WithReplicaSet(string replicaSetName = "rs0")` extension method on `IResourceBuilder<MongoDBServerResource>` that configures the MongoDB container to start as a single-node replica set
- Add `ReplicaSetName` property (public get, internal set) to `MongoDBServerResource`
- Update `BuildConnectionString()` to append `directConnection=true` to the connection string when replica set mode is enabled
- Replace the default MongoDB health check with a replica-set-aware implementation that initiates the replica set and gates readiness on primary election
- Inject a deterministic keyfile into the container (required by MongoDB when running `--replSet` with authentication)

## Capabilities

### New Capabilities

- `mongodb-replica-set`: Single-node replica set mode for the MongoDB container resource, covering the `WithReplicaSet` API, keyfile injection, container args, connection string behaviour, and health check gating

### Modified Capabilities

(none — no existing spec files to update)

## Impact

**Code**
- `src/Aspire.Hosting.MongoDB/MongoDBServerResource.cs` — new property, updated connection string builder
- `src/Aspire.Hosting.MongoDB/MongoDBBuilderExtensions.cs` — new extension method, modified health check registration
- `src/Aspire.Hosting.MongoDB/api/Aspire.Hosting.MongoDB.cs` — public API surface snapshot must be updated

**Tests**
- `tests/Aspire.Hosting.MongoDB.Tests/AddMongoDBTests.cs` — unit tests for new method and connection string behaviour
- `tests/Aspire.Hosting.MongoDB.Tests/MongoDBPublicApiTests.cs` — public API surface tests
- `tests/Aspire.Hosting.MongoDB.Tests/MongoDbFunctionalTests.cs` — functional test for replica set end-to-end

**Dependencies**
- No new NuGet dependencies; `MongoDB.Driver` (already referenced) provides `BsonDocument`, `MongoCommandException`, and the admin command API needed for replica set initiation
