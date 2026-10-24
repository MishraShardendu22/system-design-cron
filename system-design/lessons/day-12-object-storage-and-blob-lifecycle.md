# DAY 12: Object Storage and Blob Lifecycle

**Progression level:** Level 2, Storage and request-path mechanics  
**Concept ID:** SD-012  
**Prerequisites:** SD-007, Data modeling and access patterns, covered.  
**Status:** Lesson delivered; exercises pending.

### 1. Why It Exists

Large binary objects have a different access and failure profile from transactional records. Images, model artifacts, backups, video segments, archives, and user uploads can be megabytes to terabytes, while their metadata may be only a few hundred bytes. Storing the bytes directly in a relational row can increase backup volume, replication traffic, transaction duration, and database resource contention. Object storage separates the byte stream from the metadata and exposes object-oriented operations suited to large payloads.

That separation introduces a lifecycle problem. The metadata database and object store do not share a transaction. An upload can succeed while metadata finalization fails; metadata can exist while the object is missing; a delete can remove one side but not the other. Multipart uploads can be abandoned, clients can retry completion, and a retention policy can conflict with a deletion request. A design that only implements PUT and GET is incomplete.

The system must define who may upload and download, how object identity is generated, which integrity checks are enforced, how an upload becomes visible, how abandoned work is reconciled, and when data transitions to archival or deletion. Lifecycle is a correctness contract, not just a storage-cost setting.

### 2. Core Idea

Treat object storage as the data plane for bytes and a transactional metadata store as the control plane for ownership, state, authorization references, and lifecycle policy.

A typical design follows these rules:

1. Store object bytes in object storage and store searchable business metadata in a database.
2. Generate opaque object keys server-side. Do not use an untrusted filename as the storage identity.
3. Authorize an upload before issuing a short-lived upload capability.
4. Verify the completed object’s existence, size, and required checksum before marking metadata available.
5. Make lifecycle transitions explicit and idempotent.
6. Reconcile partial failures with retryable workers and periodic orphan detection.
7. Apply retention, archival, deletion, and legal-hold policies to every relevant object version.
8. Test failure between every cross-system step.

Object stores generally expose buckets or containers, keys, object metadata, versioning or generations, and lifecycle rules, but exact consistency, checksum, retention, multipart, and restore semantics vary by provider and configuration. Read the selected provider’s current contract rather than assuming a portable guarantee.

Official references: [Amazon S3 multipart uploads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html), [Amazon S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html), [Amazon S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html), [Google Cloud Storage object lifecycle management](https://cloud.google.com/storage/docs/lifecycle), and [Azure Blob Storage lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview).

### 3. Mental Model

Think of the object system as three cooperating components:

- **Control plane:** authenticates the caller, checks tenant ownership and policy, creates upload intents, stores metadata, and decides whether an object is visible.
- **Data plane:** transfers bytes directly between the client and object storage using scoped, expiring capabilities.
- **Lifecycle workers:** complete or abort multipart work, verify objects, transition retention state, delete expired data, and reconcile inconsistencies.

The database state is the application’s visibility contract. An object may physically exist but remain unavailable to normal reads until validation and metadata finalization succeed. Conversely, a database row must not be treated as proof that the object bytes exist.

A useful state machine is:

~~~mermaid
stateDiagram-v2
  [*] --> PendingUpload
  PendingUpload --> Available: object verified and finalized
  PendingUpload --> Failed: permanent validation failure
  PendingUpload --> Expired: upload intent timed out
  Available --> Deleting: deletion accepted
  Deleting --> Deleted: object deletion confirmed
  Deleting --> Deleting: retry transient failure
  Available --> Archived: lifecycle policy transition
  Archived --> Available: restore completed
  Failed --> [*]
  Expired --> [*]
  Deleted --> [*]
~~~

The exact states depend on the product, but they must answer: can the caller read the object now, who owns the next transition, and what action is safe to retry?

### 4. Real-World Example

Consider a multi-tenant artifact service used by build pipelines and ML jobs. A job uploads a model checkpoint or build artifact, then shares a reference with another internal service. Artifacts can be several gigabytes. Uploads may be interrupted, workers may crash, and artifacts must expire after a retention period unless a release or compliance policy pins them.

The API creates an upload intent with an expected byte count, media type, checksum algorithm, and retention class. The client uploads directly to object storage. The API verifies the completed object and atomically changes the metadata record from pending to available. Consumers receive a download capability only after tenant authorization and state checks. A lifecycle worker moves cold artifacts to a cheaper storage class or deletes expired objects according to policy.

**Documented facts:** Provider documentation describes multipart uploads, expiring signed requests, lifecycle transitions, and provider-specific checksum and retention behavior. The linked official references above define the relevant product contracts.

**Established industry practice:** Keep large byte transfers off application request workers where possible, enforce bounded capability lifetimes, use server-generated keys, verify integrity, and reconcile operations spanning independent systems.

**Engineering inference for this service:** A relational metadata table plus object storage is a reasonable baseline because ownership, visibility, and retention policy require transactional updates while the artifact bytes need a separate scalable data plane. The actual bucket layout, storage class, multipart threshold, retention duration, and transfer limits must be selected from measured workload, compliance requirements, and provider behavior.

### 5. Low-Level Design (LLD)

#### Metadata model

A metadata record can contain:

| Field | Purpose |
|---|---|
| object_id | Opaque API identifier, independent of the user filename |
| tenant_id | Authorization and isolation boundary |
| object_key | Server-generated storage key |
| state | pending_upload, available, deleting, deleted, failed, expired, or archived |
| expected_size | Size declared when the upload intent was created |
| actual_size | Size observed after upload completion |
| checksum_algorithm | Explicit algorithm such as SHA-256 |
| expected_checksum | Optional client-declared checksum, validated before availability |
| verified_checksum | Checksum confirmed by the chosen verification path |
| content_type | Validated media type, not blindly trusted for safe rendering |
| created_at, upload_expires_at, expires_at | Upload-intent and retention deadlines |
| version | Optimistic concurrency field for lifecycle transitions |
| storage_generation | Provider version or generation identifier when applicable |

Example relational schema:

~~~sql
CREATE TABLE stored_objects (
    object_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    object_key TEXT NOT NULL UNIQUE,
    state TEXT NOT NULL CHECK (
        state IN (
            'pending_upload', 'available', 'deleting',
            'deleted', 'failed', 'expired', 'archived'
        )
    ),
    expected_size BIGINT NOT NULL CHECK (expected_size >= 0),
    actual_size BIGINT,
    checksum_algorithm TEXT NOT NULL,
    expected_checksum TEXT,
    verified_checksum TEXT,
    content_type TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    upload_expires_at TIMESTAMPTZ NOT NULL,
    expires_at TIMESTAMPTZ NOT NULL,
    version BIGINT NOT NULL DEFAULT 1,
    storage_generation TEXT
);

CREATE INDEX stored_objects_tenant_state_expiry
    ON stored_objects (tenant_id, state, expires_at);
~~~

The schema is illustrative. Enforce allowed checksum algorithms and size limits at the service boundary as well as through database constraints where practical. A real implementation should also record policy version, encryption requirements, legal hold, and deletion or restore audit information when those are part of the contract.

#### Upload protocol

1. The authenticated client requests an upload intent with size, content type, checksum information, and intended retention class.
2. The service validates tenant quota, maximum size, media-type policy, and retention policy. It creates a pending metadata row and a random, tenant-scoped object key.
3. The service returns a short-lived upload capability restricted to the intended key and allowed operation. The client transfers bytes directly to storage.
4. For multipart upload, the client completes the provider protocol. The service or a trusted completion handler obtains object metadata and validates the expected size and checksum.
5. A conditional database update changes pending_upload to available. The update must be idempotent and must not overwrite a terminal state or a newer lifecycle version.
6. Download requests resolve object_id through metadata, check tenant authorization and state, and only then issue a short-lived read capability or stream the bytes through an authorized service.

A signed URL is a bearer capability. Anyone who obtains it may be able to use it until it expires, within its provider-defined scope. Do not log full signed URLs, place them in analytics events, or assume they are single-use unless the provider explicitly guarantees that behavior.

#### Cross-system failure matrix

| Failure point | Possible residue | Recovery |
|---|---|---|
| Metadata intent created, capability generation fails | Pending row without upload | Mark failed or let a bounded expiry worker close it |
| Client uploads bytes but never completes | Object or incomplete multipart upload | Expire intent and abort multipart work; reconcile unreferenced objects |
| Object upload succeeds, metadata finalization fails | Physical object hidden from application reads | Retry finalization after verification; retain pending state until resolved |
| Completion request is retried | Duplicate completion attempt | Return current state if already available; otherwise revalidate and use conditional transition |
| Delete state commits, object delete fails | Metadata says deleting, bytes remain | Retry idempotent deletion with backoff and alert on age |
| Object delete succeeds, final metadata update fails | Bytes absent, row remains deleting | Retry delete, treat provider “not found” as success when semantics permit, then finalize |
| Lifecycle rule transitions or expires an object | Metadata may be stale | Align database policy with provider rules and reconcile actual object state |

There is no atomic transaction across the metadata database and object store. Use explicit state, conditional updates, idempotent operations, and reconciliation rather than pretending a sequence of API calls is atomic.

### 6. High-Level Design (HLD)

~~~mermaid
flowchart LR
  Client --> API[Authenticated API]
  API --> Policy[Quota, tenant, content and retention policy]
  Policy --> DB[(Metadata database)]
  API --> Signer[Capability signer]
  Signer --> Client
  Client -->|direct upload| Store[(Object storage)]
  Store --> Verify[Completion verifier]
  Verify --> DB
  API -->|authorized download capability| Client
  DB --> Worker[Lifecycle and reconciliation workers]
  Worker --> Store
  Worker --> DB
  Worker --> Metrics[Metrics, audit and alerts]
~~~

The application control plane remains responsible for authorization and visibility. Large transfers go directly to storage when the client and threat model permit it. Lifecycle workers use a durable claim or lease mechanism so multiple instances can process work safely. The worker must not hold a database transaction open while performing a potentially slow remote storage operation.

For deletion, first commit a conditional transition to deleting and enqueue or expose durable work. Then delete the exact object key and version or generation, and finally mark the metadata deleted. If the worker crashes after the remote delete but before the final database update, retrying must converge to the same state.

### 7. Realistic Go Code

The following service layer illustrates upload intent creation and completion verification. Provider-specific adapters implement signing, object inspection, and database compare-and-swap. The example deliberately keeps provider SDK details behind interfaces because checksum fields, multipart behavior, and generation semantics differ.

~~~go
package objectlifecycle

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "errors"
    "fmt"
    "strings"
    "time"
)

var (
    ErrInvalidInput = errors.New("invalid object request")
    ErrNotFound = errors.New("object not found")
    ErrConflict = errors.New("object lifecycle conflict")
    ErrIntegrity = errors.New("object integrity validation failed")
    ErrNotAvailable = errors.New("object is not available")
)

type State string

const (
    StatePendingUpload State = "pending_upload"
    StateAvailable    State = "available"
    StateDeleting     State = "deleting"
    StateDeleted      State = "deleted"
    StateFailed       State = "failed"
    StateExpired      State = "expired"
)

type Object struct {
    ID                string
    TenantID          string
    Key               string
    State             State
    ExpectedSize      int64
    ActualSize        int64
    ChecksumAlgorithm string
    ExpectedChecksum  string
    VerifiedChecksum  string
    ContentType       string
    UploadExpiresAt   time.Time
    ExpiresAt         time.Time
    StorageGeneration string
    Version           int64
}

type UploadRequest struct {
    TenantID          string
    Size              int64
    ContentType       string
    ChecksumAlgorithm string
    ExpectedChecksum  string
    Retention         time.Duration
}

type UploadIntent struct {
    ObjectID  string
    UploadURL string
    ExpiresAt time.Time
}

type ObjectInfo struct {
    Size              int64
    ContentType       string
    ChecksumAlgorithm string
    Checksum          string
    Generation        string
}

type Repository interface {
    InsertPending(ctx context.Context, object Object) error
    GetByTenantAndID(ctx context.Context, tenantID, objectID string) (Object, error)
    MarkAvailable(ctx context.Context, objectID string, expectedVersion int64, info ObjectInfo) (Object, error)
}

type BlobStore interface {
    CreateUploadURL(ctx context.Context, key, contentType string, expiresAt time.Time) (string, error)
    Head(ctx context.Context, key string) (ObjectInfo, error)
}

type Service struct {
    repo       Repository
    store      BlobStore
    now        func() time.Time
    maxSize    int64
    uploadTTL  time.Duration
    maxRetain  time.Duration
}

func NewService(repo Repository, store BlobStore, now func() time.Time) (*Service, error) {
    if repo == nil || store == nil {
        return nil, errors.New("repository and blob store are required")
    }
    if now == nil {
        now = time.Now
    }
    return &Service{
        repo: repo, store: store, now: now,
        maxSize: 20 << 30,
        uploadTTL: 15 * time.Minute,
        maxRetain: 365 * 24 * time.Hour,
    }, nil
}

func newID() (string, error) {
    var b [16]byte
    if _, err := rand.Read(b[:]); err != nil {
        return "", fmt.Errorf("generate object id: %w", err)
    }
    return hex.EncodeToString(b[:]), nil
}

func validContentType(value string) bool {
    switch strings.ToLower(strings.TrimSpace(value)) {
    case "application/octet-stream", "application/zip", "application/pdf",
        "application/json", "application/x-tar":
        return true
    default:
        return false
    }
}

func validChecksum(algorithm, checksum string) bool {
    if checksum == "" {
        return algorithm == "none"
    }
    if algorithm != "sha256" || len(checksum) != 64 {
        return false
    }
    _, err := hex.DecodeString(checksum)
    return err == nil
}

func (s *Service) CreateUpload(ctx context.Context, req UploadRequest) (UploadIntent, error) {
    if ctx == nil ||
        strings.TrimSpace(req.TenantID) == "" ||
        req.Size < 0 || req.Size > s.maxSize ||
        !validContentType(req.ContentType) ||
        !validChecksum(req.ChecksumAlgorithm, req.ExpectedChecksum) ||
        req.Retention <= 0 || req.Retention > s.maxRetain {
        return UploadIntent{}, ErrInvalidInput
    }

    id, err := newID()
    if err != nil {
        return UploadIntent{}, err
    }

    now := s.now().UTC()
    uploadExpiresAt := now.Add(s.uploadTTL)
    expiresAt := now.Add(req.Retention)
    key := "objects/" + id
    object := Object{
        ID: id, TenantID: req.TenantID, Key: key,
        State: StatePendingUpload, ExpectedSize: req.Size,
        ChecksumAlgorithm: req.ChecksumAlgorithm,
        ExpectedChecksum: req.ExpectedChecksum,
        ContentType: req.ContentType,
        UploadExpiresAt: uploadExpiresAt,
        ExpiresAt: expiresAt, Version: 1,
    }
    if err := s.repo.InsertPending(ctx, object); err != nil {
        return UploadIntent{}, fmt.Errorf("persist upload intent: %w", err)
    }

    capabilityExpiry := uploadExpiresAt
    uploadURL, err := s.store.CreateUploadURL(ctx, key, req.ContentType, capabilityExpiry)
    if err != nil {
        return UploadIntent{}, fmt.Errorf("create upload capability: %w", err)
    }
    return UploadIntent{ObjectID: id, UploadURL: uploadURL, ExpiresAt: capabilityExpiry}, nil
}

func (s *Service) CompleteUpload(ctx context.Context, tenantID, objectID string) (Object, error) {
    if ctx == nil || strings.TrimSpace(tenantID) == "" || strings.TrimSpace(objectID) == "" {
        return Object{}, ErrInvalidInput
    }

    object, err := s.repo.GetByTenantAndID(ctx, tenantID, objectID)
    if err != nil {
        return Object{}, fmt.Errorf("load upload intent: %w", err)
    }
    if object.State == StateAvailable {
        return object, nil
    }
    if object.State != StatePendingUpload || !s.now().Before(object.UploadExpiresAt) {
        return Object{}, ErrConflict
    }

    info, err := s.store.Head(ctx, object.Key)
    if err != nil {
        return Object{}, fmt.Errorf("inspect uploaded object: %w", err)
    }
    if info.Size != object.ExpectedSize ||
        info.ContentType != object.ContentType {
        return Object{}, ErrIntegrity
    }
    if object.ChecksumAlgorithm != "none" &&
        (info.ChecksumAlgorithm != object.ChecksumAlgorithm ||
            !strings.EqualFold(info.Checksum, object.ExpectedChecksum)) {
        return Object{}, ErrIntegrity
    }

    available, err := s.repo.MarkAvailable(ctx, object.ID, object.Version, info)
    if err != nil {
        if errors.Is(err, ErrConflict) {
            current, readErr := s.repo.GetByTenantAndID(ctx, tenantID, objectID)
            if readErr == nil && current.State == StateAvailable {
                return current, nil
            }
        }
        return Object{}, fmt.Errorf("finalize upload: %w", err)
    }
    return available, nil
}
~~~

Important adapter contract:

- InsertPending must enforce unique object IDs and keys.
- GetByTenantAndID must scope the lookup by authenticated tenant, not just object ID.
- MarkAvailable must atomically require the expected version and pending_upload state, then persist actual size, verified checksum, generation, state, and incremented version.
- The BlobStore adapter must return a trustworthy checksum for the requested algorithm. If the provider does not expose a suitable verified checksum for the upload mode, the service must use an explicit verification path instead of treating ETag as a universal content hash.
- Capability generation failure leaves a pending intent; an expiry worker must close it. Do not silently assume the database insert was rolled back.
- A production implementation should validate the checksum encoding and algorithm exactly, use provider-native multipart completion when applicable, and map provider errors into retryable, permanent, and not-found classes.

### 8. Example Usage

~~~go
ctx, cancel := context.WithTimeout(request.Context(), 3*time.Second)
defer cancel()

intent, err := objectService.CreateUpload(ctx, objectlifecycle.UploadRequest{
    TenantID:          authenticatedTenantID,
    Size:              declaredSize,
    ContentType:       validatedContentType,
    ChecksumAlgorithm: "sha256",
    ExpectedChecksum:  declaredSHA256,
    Retention:         30 * 24 * time.Hour,
})
if err != nil {
    return err
}

writeJSON(response, map[string]any{
    "object_id": intent.ObjectID,
    "upload_url": intent.UploadURL,
    "expires_at": intent.ExpiresAt,
})
~~~

After the client finishes the provider upload protocol, it calls the completion endpoint with the opaque object ID. The service derives the tenant from authenticated identity, inspects the object, and conditionally finalizes metadata. The client must not be allowed to choose an arbitrary storage key or declare the object available without server-side verification.

The upload URL is intentionally short-lived. Avoid returning a permanent bucket path or broad credentials. If a client needs multipart upload, the API should issue only the required part capabilities and persist enough upload-session information to complete or abort the session safely.

### 9. Production Considerations

- **Integrity:** Define the checksum algorithm and exact bytes covered. Multipart ETags are not a portable substitute for a full-object cryptographic checksum. Verify provider behavior for the selected upload mode.
- **Size enforcement:** Enforce declared and observed sizes. Where supported, constrain uploads at the storage boundary as well as checking them after upload.
- **Capability security:** Use short expiration, least privilege, TLS, careful URL redaction, and strict tenant/object binding. A signed URL is a bearer secret.
- **Key design:** Use opaque, server-generated keys. Normalize tenant identifiers and prevent path confusion. User filenames belong in validated metadata, not as trusted storage identity.
- **Visibility:** Serve only objects in an allowed state. Physical existence does not mean the application has accepted the object.
- **Multipart cleanup:** Track or discover incomplete multipart uploads and abort them after a bounded timeout. Incomplete parts may incur storage cost.
- **Reconciliation:** Periodically find old pending intents, unreferenced objects, missing objects, stale deleting rows, and provider lifecycle actions that diverge from metadata. Make repair actions auditable and rate-limited.
- **Delete semantics:** Delete the intended version or generation when versioning is enabled. Treat provider not-found as success only when it proves the requested object/version is absent under the provider contract.
- **Retention and legal hold:** Separate user-requested deletion from policy-mandated retention. A legal hold or retention lock may make deletion impossible until the policy allows it.
- **Lifecycle policies:** Align provider lifecycle rules with database retention. Object expiration that runs independently can leave stale metadata; metadata cleanup that runs first can orphan bytes. Use grace periods and reconciliation.
- **Archive restore:** Cold storage may require a restore request and waiting period. Model archived or restore-pending state explicitly and expose expected delay to callers.
- **Versioning:** Decide whether overwrites create new versions or are prohibited. Keep metadata references to the exact version/generation when reproducibility or legal evidence matters.
- **Access control:** Keep buckets private by default. Use tenant-scoped authorization before issuing read capabilities. Consider malware scanning, content sniffing, and safe content-disposition headers for untrusted uploads.
- **Encryption:** Define encryption at rest, key ownership, rotation, and how key deletion affects recoverability.
- **Observability:** Track upload intent count and age, completion success, checksum mismatch, orphan count, multipart bytes, delete backlog, lifecycle lag, restore duration, storage by class, and cost by tenant.
- **Timeouts and retries:** Use bounded contexts and exponential backoff with jitter for remote calls. Retry idempotent operations; do not repeat an ambiguous multipart completion without checking provider state.
- **Capacity and cost:** Estimate bytes stored, request rates, multipart overhead, retrieval and egress, lifecycle transitions, replication, and restore charges. The cheapest storage class is not cheapest if retrieval latency or fees violate the workload.
- **Recovery:** Test database restore and object restore together. A metadata backup without corresponding object versions, or objects without recoverable metadata, may not restore the service correctly.
- **Audit:** Record who created, downloaded, retained, restored, or deleted an object, while avoiding sensitive capability URLs and content leakage in logs.

### 10. Tradeoffs

| Design choice | Benefit | Cost or risk |
|---|---|---|
| Direct client-to-object upload | Keeps large transfers off API workers and scales data throughput separately | Requires scoped capabilities, completion verification, and reconciliation |
| Proxy bytes through the API | Central inspection and simpler client protocol | API bandwidth, connection occupancy, memory/disk pressure, and additional failure surface |
| Store bytes in relational database | Transactional coupling for small payloads | Database growth, backup/replication cost, and contention for large objects |
| Metadata database plus object store | Independent scaling and explicit object lifecycle | No cross-system atomic transaction; requires state machine and repair |
| Immutable object keys | Simplifies integrity, caching, and version references | Replacements create new objects and require metadata switching and cleanup |
| In-place overwrite | Simple logical name | Races, cache invalidation, version ambiguity, and harder rollback |
| Provider lifecycle rules | Efficient automated tiering and expiry | Provider action can diverge from application metadata unless policies are aligned |
| Application-managed lifecycle | Rich business policy and auditability | Worker operations, retries, backlog monitoring, and more control-plane complexity |

Select direct upload when payload size and throughput justify it and the security model supports signed capabilities. Proxying may be appropriate when every byte must pass through a mandatory inspection pipeline, but that pipeline should be designed as a streaming service rather than buffering arbitrary payloads in request memory.

### 11. Connection to Previous Lessons

Day 7, SD-007, established entity lifecycle, ownership, and query-driven metadata design. Day 8, SD-008, established transactional state changes. Day 11, SD-011, emphasized matching storage models to access patterns and acknowledging product-specific guarantees. SD-012 applies those principles to a split data plane where metadata transactions and byte operations cannot commit atomically together.

The metadata table is not the blob store, and the object store is not the application’s complete source of authorization truth. Correctness comes from the protocol and recovery logic between them.

### 12. What Comes Next

**SD-013: Caching Fundamentals and Cache Policies**, Level 2. It covers cache-aside, TTLs, eviction, cache keys, and the tradeoff between origin load and stale data. Its prerequisites, SD-004 and SD-007, are covered. SD-015 and SD-019 are also eligible, but SD-013 is next in curriculum order. It is not taught here.

### 13. Mini Exercise

A multi-tenant build platform stores 8 million artifacts. Median artifact size is 18 MB, the 99th percentile is 4 GB, and clients upload through unreliable networks. Metadata must become visible only after integrity verification. Artifacts expire after 45 days unless pinned by a release. Deletion must respect a legal hold. The current design streams all bytes through API servers and writes the metadata row before upload.

Design an upload and lifecycle protocol. Specify metadata states, the upload capability scope and lifetime, multipart handling, checksum verification, and the point at which the artifact becomes visible. Explain how you recover from a crash after bytes are stored but before metadata finalization, and after deletion succeeds but metadata still says deleting. Include a retention and legal-hold policy, plus metrics and failure tests.

**Acceptance criteria:** Show the state machine, identify the cross-system consistency gap, define idempotent retry behavior, distinguish integrity verification from ETag assumptions, and describe a reconciliation strategy with bounded cost and auditable actions.

### Key Takeaways

- Keep large object bytes separate from transactional metadata when their scaling and access patterns differ.
- Treat upload completion and metadata visibility as an explicit protocol.
- Verify size and a defined checksum before marking an object available.
- Do not assume the metadata database and object store share an atomic transaction.
- Use opaque keys, tenant-scoped authorization, and short-lived least-privilege capabilities.
- Make completion, deletion, expiry, and reconciliation idempotent.
- Align provider lifecycle policies with metadata state, retention, versioning, and legal holds.
- Test partial failure, restore, and orphan recovery, not only the successful upload path.

### Concept Map

~~~mermaid
flowchart TD
  A[SD-007 Data ownership and lifecycle] --> B[SD-012 Object storage and blob lifecycle]
  C[Metadata control plane] --> B
  D[Object byte data plane] --> B
  B --> E[Upload intent and scoped capability]
  B --> F[Checksum and completion verification]
  B --> G[Retention, archive and deletion]
  B --> H[Reconciliation and recovery]
  B --> I[SD-054 Media ingestion and delivery]
  B --> J[SD-046 Data protection and threat modeling]
  B --> K[SD-013 Caching fundamentals]
~~~

### Exercise

Design an artifact storage service for a multi-tenant build or ML platform.

**Deliverables:** Metadata schema and lifecycle state machine; API contract for upload intent, completion, download, pinning, and deletion; Go service and provider adapter boundaries; checksum and size verification; multipart completion and abort handling; worker logic for expiry and deletion; reconciliation queries and repair policy; integration tests covering crashes and duplicate requests; load estimates for storage, request rate, egress, and lifecycle operations; observability and audit fields; and a restore rehearsal.

**Acceptance criteria:** Demonstrate tenant isolation, no premature visibility, atomic metadata state transitions, idempotent completion and deletion, retention/legal-hold enforcement, detection of orphaned or missing objects, and recovery after a process crash at each cross-system boundary. Pending until submitted and reviewed.

### Progress

- **Day:** 12, following the persisted Day 1 through Day 11 lesson history on main.
- **Progression level:** Level 2, Storage and request-path mechanics.
- **Current concept:** SD-012, Object storage and blob lifecycle.
- **Prerequisite satisfied:** SD-007.
- **Mini exercise:** Pending learner submission.
- **Substantial exercise:** Pending learner submission.
- **Concepts covered after persistence:** SD-001 through SD-012.
- **Next concept:** SD-013, Caching fundamentals and cache policies.
- **Other eligible concepts:** SD-015, Proxies, reverse proxies, and API gateways; SD-019, Replication fundamentals.
- **Repository update status:** Lesson, curriculum status, and progress record must be committed together and read back from main before reporting completion.

### Most Important Mental Model

**An object is usable only when its bytes, verified integrity, metadata state, authorization, and lifecycle policy agree. Design the protocol to converge safely when any cross-system step fails.**
