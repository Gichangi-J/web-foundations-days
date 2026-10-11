SnapShare: Photo App Scaling Plan

1. Assumptions

SnapShare is a photo-sharing app where users upload photos and scroll through feeds containing photos from people they follow.

The following assumptions are used:

- Registered users: 10,000,000.
- Daily active users: 10% of registered users.
- Each active user uploads 1 photo per day.
- Each active user views 50 feed pages per day.
- Average original photo size: 2 MB.
- Average thumbnail size: 50 KB.
- One year has 365 days.
- A day has 86,400 seconds.
- Peak traffic is estimated at 5 times the average traffic.
- Storage estimates exclude replication, backups, and other database data.
- All uploaded photos are retained for at least one year.

2. Traffic and Storage Estimates

Daily Active Users

Daily active users = 10,000,000 × 10%

Daily active users = 1,000,000
Uploads Per Second

Each active user uploads one photo per day.

Daily uploads = 1,000,000 × 1 = 1,000,000 photos.

Average uploads per second = 1,000,000 ÷ 86,400

Average uploads per second ≈ 11.57 photos

Peak uploads per second = 11.57 × 5

Peak uploads per second ≈ 57.87 photos

Feed Views Per Second

Each active user views 50 feed pages per day.

Daily feed views = 1,000,000 × 50 = 50,000,000 feed pages.

Average feed views per second = 50,000,000 ÷ 86,400

Average feed views per second ≈ 578.70 feed pages

Peak feed views per second = 578.70 × 5

Peak feed views per second ≈ 2,893.52 feed pages

Photo Storage Per Year

Daily original photo storage = 1,000,000 × 2 MB = 2,000,000 MB, or approximately 2 TB per day using decimal units.

Annual original photo storage = 2 TB × 365

Annual original photo storage ≈ 730 TB

Daily thumbnail storage = 1,000,000 × 50 KB = 50,000,000 KB, or approximately 50 GB per day.

Annual thumbnail storage = 50 GB × 365

Annual thumbnail storage ≈ 18.25 TB

Total annual storage for originals and thumbnails:

730 TB + 18.25 TB = 748.25 TB per year

These are decimal storage estimates before replication, backups, and additional overhead. If photo uploads are not deleted, storage requirements will accumulate year after year.

3. Read-Heavy or Write-Heavy?

SnapShare is a read-heavy system because users request 50 feed pages per day but upload only one photo per day. This produces approximately 50 million feed views compared with 1 million uploads daily.

The architecture should therefore prioritize fast feed delivery using a CDN, caching, and database read replicas. Upload processing should also be reliable, but the system needs to handle significantly more feed requests than new uploads.

4. Where Photos Should Be Stored

Photo files should be stored in object storage, such as an S3-compatible storage service, rather than directly inside the relational database.

Storing large image files in the database increases database storage requirements, backup sizes, and the cost of transferring data. Object storage is designed for large files and can scale independently from the database.

The database should store photo metadata, such as the photo ID, owner ID, caption, upload timestamp, and object-storage key. The original photo and thumbnail are stored as objects, and their keys are referenced by the database.

5. Architecture Diagram

```text
                         USERS
                           |
                           v
                     +-----------+
                     |    CDN    |
                     +-----------+
                       |       |
              Cached photos   Cache miss / API request
                       |       |
                       |       v
                       |  +----------------+
                       |  | Load Balancer  |
                       |  +----------------+
                       |          |
                       |          v
                       |  +----------------+
                       |  |  App Servers   |
                       |  +----------------+
                       |    |     |      |
                       |    |     |      +------------------+
                       |    |     |                         |
                       |    v     v                         v
                       |  +-------+                +----------------+
                       |  | Cache |                | Message Queue  |
                       |  +-------+                +----------------+
                       |    |                            |
                       |    v                            v
                       |  +----------------+      +----------------+
                       |  | Primary DB     |      | Thumbnail      |
                       |  | (writes)       |      | Worker         |
                       |  +----------------+      +----------------+
                       |    |                            |
                       |    v                            v
                       |  +----------------+      +----------------+
                       |  | Read Replica   |      | Object Storage |
                       |  | (feed reads)   |      | Originals and  |
                       |  +----------------+      | thumbnails     |
                       |                           +----------------+
                       |                                  |
                       +----------------------------------+
                            Photos served through CDN
```

6. Components and the Problems They Solve

-CDN: Caches and delivers photos close to users, reducing latency and traffic reaching the application servers and object storage.
-Load balancer: Distributes incoming requests across application servers so that one server does not become a bottleneck.
-Application servers: Handle authentication, uploads, feed requests, permissions, and communication between system components.
-Cache: Keeps frequently accessed feed data and metadata in fast memory, reducing repeated database queries.
-Primary database: Stores authoritative records such as users, follows, photo metadata, and upload information.
-Read replica: Serves read-heavy feed queries to reduce load on the primary database, while accepting that replication may introduce brief delays.
-Object storage: Stores original photos and thumbnails durably without filling the database with large binary files.
-Message queue: Holds thumbnail-generation jobs so that photo uploads do not need to wait for image processing to finish.
Thumbnail worker: Processes queued jobs to create smaller versions of uploaded photos and saves those thumbnails to object storage.

7. Photo Upload Flow

1. A user selects a photo in the SnapShare app and submits it to the application.
1. The application authenticates the user and validates the file type, size, and upload permissions.
1. The original photo is uploaded to object storage, using a unique object key.
1. The application records the photo's metadata and object key in the primary database.
1. A thumbnail-generation job is added to the message queue.
1. The application returns a successful upload response without waiting for thumbnail generation to finish.
1. A thumbnail worker retrieves the job from the queue and downloads or reads the original photo from object storage.
1. The worker creates a 50 KB thumbnail and stores it in object storage under a separate object key.
1. The worker updates the photo metadata to indicate that the thumbnail is ready, if the system tracks processing status.
1. When users scroll their feeds, application servers retrieve feed metadata and photo references, while the CDN serves cached photo files or fetches them from object storage when needed.

If thumbnail processing fails, the job can be retried. Failed jobs should eventually be recorded for investigation rather than retried indefinitely.

8. Trade-Offs

Trade-Off 1: Faster Reads vs. Freshness

Using a cache and read replica makes feeds faster and reduces primary database load. However, cached results may be outdated, and a read replica may lag behind the primary database. The application should use cache expiration or invalidation and read from the primary database when a fresh result is essential.

Trade-Off 2: Asynchronous Thumbnails vs. Immediate Availability

Using a queue allows uploads to finish quickly and lets thumbnail processing scale independently. However, thumbnails may not be ready immediately after an upload. The application can show a placeholder until the thumbnail is available and retry failed jobs.

Trade-Off 3: Object Storage vs. Simplicity

Object storage scales well for large photo files and separates file storage from database operations. However, the system must manage object keys, access permissions, upload failures, and cleanup of files that are no longer referenced by database records.

Trade-Off 4: Replication and Availability vs. Cost

Running multiple application servers, a read replica, a cache, a queue, and durable object storage improves scalability and resilience. However, these components increase infrastructure cost and operational complexity. They should be monitored and scaled according to actual traffic and service requirements.

Conclusion

SnapShare should use a read-optimized architecture with a CDN, cache, application servers, and a database read replica to handle feed traffic. Original photos and thumbnails belong in object storage, while a queue and background workers handle thumbnail generation. Based on the assumptions, the system expects approximately 579 average feed views per second, 2,894 peak feed views per second, and 748.25 TB of new photo and thumbnail storage per year.
