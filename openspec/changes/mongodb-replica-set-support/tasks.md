## 1. MongoDBServerResource — property and connection string

- [x] 1.1 Add `public string? ReplicaSetName { get; internal set; }` property to `MongoDBServerResource`
- [x] 1.2 Update `BuildConnectionString(string? databaseName)` to append `&directConnection=true` when `ReplicaSetName` is set and a password is present, or `?directConnection=true` when no password is present
- [x] 1.3 Update `GetConnectionProperties()` to yield `DirectConnection = "true"` when `ReplicaSetName` is set

## 2. Health check — replica-set-aware implementation

- [x] 2.1 Define an internal sealed `MongoDBServerHealthCheck` class implementing `IHealthCheck`, accepting the `MongoDBServerResource` and a `Func<string?>` connection string provider
- [x] 2.2 Implement the standalone (non-replica-set) path in `CheckHealthAsync`: call `client.ListDatabaseNames()` and return `Healthy`
- [x] 2.3 Implement the replica set path: attempt `replSetInitiate` using `mongoDBContainer.Name:27017` as the member host; catch error code 23 (`AlreadyInitialized`); gate healthy return on `isWritablePrimary: true`
- [x] 2.4 Make the `_replicaSetInitiated` flag thread-safe using `Interlocked.CompareExchange`; reset to 0 on unexpected initiation failures so the next health check retries
- [x] 2.5 In `AddMongoDB`, replace the `AddMongoDb()` extension call with a `HealthCheckRegistration` that instantiates `MongoDBServerHealthCheck`; preserve the cached `IMongoClient` across invocations

## 3. WithReplicaSet extension method

- [x] 3.1 Add `WithReplicaSet(this IResourceBuilder<MongoDBServerResource> builder, string replicaSetName = "rs0")` to `MongoDBBuilderExtensions`
- [x] 3.2 Guard: `ArgumentNullException.ThrowIfNull(builder)`, `ArgumentException.ThrowIfNullOrEmpty(replicaSetName)`, throw `InvalidOperationException` if `builder.Resource.ReplicaSetName` is already set
- [x] 3.3 Set `builder.Resource.ReplicaSetName = replicaSetName`
- [x] 3.4 Call `WithArgs("--replSet", replicaSetName, "--keyFile", "/tmp/mongodb-keyfile", "--bind_ip_all")`
- [x] 3.5 Generate deterministic keyfile: `Convert.ToBase64String(SHA256.HashData(Encoding.UTF8.GetBytes($"{builder.Resource.Name}-mongodb-keyfile")))`
- [x] 3.6 Call `WithContainerFiles("/tmp", [ new ContainerFile { Name = "mongodb-keyfile", Contents = keyFileContent, Mode = UnixFileMode.UserRead | UnixFileMode.UserWrite } ], defaultOwner: 999, defaultGroup: 999)`
- [x] 3.7 Mark `WithReplicaSet` with `[AspireExportIgnore(Reason = "Uses WithContainerFiles which is not available in polyglot app hosts.")]`

## 4. Public API surface

- [x] 4.1 Run `dotnet build` (or the API snapshot tool) to regenerate `src/Aspire.Hosting.MongoDB/api/Aspire.Hosting.MongoDB.cs`; verify `ReplicaSetName` getter and `WithReplicaSet` method appear in the snapshot

## 5. Unit tests — AddMongoDBTests

- [x] 5.1 Test: `WithReplicaSet()` sets `ReplicaSetName` to `"rs0"` on the resource
- [x] 5.2 Test: `WithReplicaSet("myset")` sets `ReplicaSetName` to `"myset"`
- [x] 5.3 Test: container args include `--replSet`, `--keyFile /tmp/mongodb-keyfile`, `--bind_ip_all` after `WithReplicaSet`
- [x] 5.4 Test: a `ContainerFileSystemCallbackAnnotation` targeting `/tmp` is present after `WithReplicaSet`
- [x] 5.5 Test: keyfile content is identical when `WithReplicaSet` is called twice on resources with the same name (determinism)
- [x] 5.6 Test: keyfile content differs between two resources with different names (uniqueness)
- [x] 5.7 Test: server connection string ends with `&directConnection=true` when `WithReplicaSet` is enabled and password is present
- [x] 5.8 Test: server connection string ends with `?directConnection=true` when `WithReplicaSet` is enabled and no password is present
- [x] 5.9 Test: database connection string includes `directConnection=true` when parent has `WithReplicaSet` enabled
- [x] 5.10 Test: connection string does NOT contain `directConnection` when `WithReplicaSet` is not called
- [x] 5.11 Test: `WithReplicaSet(null!)` throws `ArgumentException`
- [x] 5.12 Test: `WithReplicaSet("")` throws `ArgumentException`
- [x] 5.13 Test: calling `WithReplicaSet` twice throws `InvalidOperationException`
- [x] 5.14 Test: `WithReplicaSet` called on null builder throws `ArgumentNullException`

## 6. Unit tests — MongoDBPublicApiTests

- [x] 6.1 Add a public API snapshot test asserting `WithReplicaSet` is present with `[AspireExportIgnore]`
- [x] 6.2 Add a public API snapshot test asserting `ReplicaSetName` property is present on `MongoDBServerResource`

## 7. Functional test — MongoDbFunctionalTests

- [x] 7.1 Add a functional test that starts a MongoDB container with `WithReplicaSet()`, waits for it to become healthy, connects via the driver, and verifies that a multi-document transaction commits successfully
