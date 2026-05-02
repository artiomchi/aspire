## ADDED Requirements

### Requirement: WithReplicaSet configures single-node replica set mode
Calling `WithReplicaSet` on a `MongoDBServerResource` builder SHALL configure the MongoDB container to start as a single-node replica set. This includes passing `--replSet <name>`, `--keyFile /tmp/mongodb-keyfile`, and `--bind_ip_all` as container arguments, and injecting a keyfile at `/tmp/mongodb-keyfile` owned by UID 999 with permissions 0600.

#### Scenario: Default replica set name
- **WHEN** `WithReplicaSet()` is called with no arguments
- **THEN** the container SHALL be configured with `--replSet rs0`

#### Scenario: Custom replica set name
- **WHEN** `WithReplicaSet("myset")` is called with a custom name
- **THEN** the container SHALL be configured with `--replSet myset`

#### Scenario: Null builder throws
- **WHEN** `WithReplicaSet` is called with a null builder
- **THEN** an `ArgumentNullException` SHALL be thrown

#### Scenario: Null or empty replica set name throws
- **WHEN** `WithReplicaSet` is called with a null or empty string as the replica set name
- **THEN** an `ArgumentException` SHALL be thrown

#### Scenario: Duplicate call throws
- **WHEN** `WithReplicaSet` is called more than once on the same resource builder
- **THEN** an `InvalidOperationException` SHALL be thrown

---

### Requirement: Keyfile is deterministic and stable across AppHost restarts
The injected keyfile content SHALL be derived deterministically from the resource name so that it is identical across AppHost runs. This ensures persistent containers are not recreated solely because the keyfile content changed.

#### Scenario: Stable content across runs
- **WHEN** the AppHost is started multiple times with the same resource name and `WithReplicaSet` enabled
- **THEN** the injected keyfile SHALL have identical content on every run

#### Scenario: Unique content per resource name
- **WHEN** two MongoDB resources with different names both call `WithReplicaSet`
- **THEN** each resource's injected keyfile SHALL have different content

---

### Requirement: Replica set is initiated before the resource is marked healthy
The health check for a replica set-enabled MongoDB resource SHALL initiate the replica set (via `rs.initiate()`) and SHALL NOT return a healthy status until `isWritablePrimary` is `true`.

#### Scenario: Resource becomes healthy only after primary election
- **WHEN** `WithReplicaSet` is enabled and the container starts
- **THEN** the resource SHALL remain unhealthy until a primary is elected
- **THEN** the resource SHALL become healthy once `isWritablePrimary` is `true`

#### Scenario: Initiation is idempotent across health check invocations
- **WHEN** the health check runs multiple times after the replica set is already initiated
- **THEN** `rs.initiate()` SHALL NOT be called again after first successful initiation or `AlreadyInitialized` error

---

### Requirement: Connection string includes directConnection=true for replica set mode
When `ReplicaSetName` is set on a `MongoDBServerResource`, the connection string expression SHALL include `directConnection=true` as a query parameter. This applies to both server-level and database-level connection strings.

#### Scenario: Server connection string with auth and replica set
- **WHEN** `WithReplicaSet` is enabled and a password is configured
- **THEN** the server connection string SHALL end with `?authSource=admin&authMechanism=SCRAM-SHA-256&directConnection=true`

#### Scenario: Server connection string without auth and with replica set
- **WHEN** `WithReplicaSet` is enabled and no password is configured
- **THEN** the server connection string SHALL end with `?directConnection=true`

#### Scenario: Database connection string includes directConnection
- **WHEN** `WithReplicaSet` is enabled and `AddDatabase` is called
- **THEN** the database connection string SHALL also include `directConnection=true`

#### Scenario: Connection string without replica set is unchanged
- **WHEN** `WithReplicaSet` is not called
- **THEN** the connection string SHALL NOT contain `directConnection=true`

---

### Requirement: ReplicaSetName is publicly readable on the resource
`MongoDBServerResource` SHALL expose a `ReplicaSetName` property (nullable string, public get, internal set) that returns the configured replica set name, or `null` if replica set mode is not enabled.

#### Scenario: ReplicaSetName reflects configured value
- **WHEN** `WithReplicaSet("rs0")` is called
- **THEN** `resource.ReplicaSetName` SHALL equal `"rs0"`

#### Scenario: ReplicaSetName is null by default
- **WHEN** `WithReplicaSet` is not called
- **THEN** `resource.ReplicaSetName` SHALL be `null`

---

### Requirement: WithReplicaSet is excluded from the polyglot app host export
`WithReplicaSet` SHALL be marked `[AspireExportIgnore]` and SHALL NOT be available in polyglot (ATS) app hosts. The feature is scoped to .NET app hosts only.

#### Scenario: AspireExportIgnore attribute is present
- **WHEN** the public API surface of `Aspire.Hosting.MongoDB` is inspected
- **THEN** `WithReplicaSet` SHALL carry the `AspireExportIgnore` attribute with an appropriate reason
