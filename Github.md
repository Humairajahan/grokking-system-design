# Design Github

# Table of contents

- [Functional requirements](#1-functional-requirements)
- [Non-functional requirements](#2-non-functional-requirements)
- [Capacity estimation](#3-capacity-estimation)
- [Database design](#4-database-design)
- [API design](#5-api-design)
- [High-level design](#6-high-level-design)

# 1. Functional requirements

1. Users can create, update and manage their own profiles.
2. Users can create, fork and star repositories.
3. Users can create public/private repositories.
4. Users can follow other users.

Future extensions:

5. Every commit version should be stored. So that users can view or rebase to any of the previous commits.
6. Users can clone repositories locally.
7. Users can be under organizations.
8. Managing issues and pull requests.
9. Branches
10. Notifications

# 2. Non-functional requirements

1. The repository size should be strictly less than 25 MB.
2. Latency for fetching code should be kept at 10 ms.
3. System should be highly reliable. No uploaded code should ever be lost.
4. Consistency during fetching code can take a hit. If a user does not see code for a while, that is fine.

# 3. Capacity estimation

|                                                       |          |
| ----------------------------------------------------- | -------- |
| Total number of users                                 | 50M      |
| Daily active users (10%)                              | 5M       |
| Number of new repositories created per user every day | 2        |
| Read/Write ratio                                      | 10:1     |
| Average repository size                               | 1 MB     |
| Data stored for                                       | 10 years |

### QPS

- Total new repositories created every day: 5M \* 2 repos -> 10M
- Write QPS: 10M/24h/3600s -> ~115
- Peak QPS: 2 \* QPS -> 2\*115 -> 230

- Read QPS: 10 \* Write QPS -> 1150
- Peak read QPS: 2 \* Read QPS -> 2300

### Storage estimation

- Average repository size 1 MB
- Total new repositories created every day: 5M \* 2 repos -> 10M
- Data storage estimation (per day): 10M \* 1MB -> 10 GB
- Data storage estimation (for 10 years): 10GB\*365d\*10y -> 36.5 TB

# 4. Database design

| User        |        | Repository          |                               | Star          |               | Followers    |               |
| ----------- | ------ | ------------------- | ----------------------------- | ------------- | ------------- | ------------ | ------------- |
| id          | PK     | id                  | PK                            | id            | PK            | id           | PK            |
| username    | unique | owner_id            | FK to User.id                 | repository_id | FK            | follower_id  | FK to User.id |
| email       | unique | name                |                               | user_id       | FK to User.id | following_id | FK to User.id |
| password    |        | description         |                               | created_at    |               | created_at   |               |
| bio         |        | visibility_status   | ENUM: public/private          |               |               |              |               |
| avatar_url  |        | forked_from_repo_id | Nullable. FK to Repository.id |               |               |              |               |
| verified_at |        | created_at          |                               |               |               |              |               |
| created_at  |        | updated_at          |                               |               |               |              |               |
| updated_at  |        |                     |                               |               |               |              |               |

- A user can create many repositories. **User-Repository: One-to-many relationship**
- A repository can be forked by many users. **Repository-Fork: One-to-many relationship**
- Many users could star a repository. **Star: Pivot table**
- A user can follow many other users. **Followers: Pivot table**

# 5. API design

```bash
# Authentication
POST    /signup
POST    /signin
POST    /email-verification
POST    /signout
POST    /password-reset

# User Profile
GET     /user/{username}
PATCH   /user
POST    /user/follow?target=username
GET     /user/username/followers
GET     /user/username/following

# Repository
POST    /repository
    {
        username: a
        repository_name: ..
        visibility_status: public/private
    }
GET     /username/repository_name
PATCH   /username/repository_name
DELETE  /username/repository_name
GET     /user/username/repositories
GET     /repositories/search?q=...

# Code
GET     /username/repository_name/..

# Fork
POST    /username/repository_name/fork
DELETE  /username/repository_name/fork
GET     /username/repository_name/forks

# Star
POST    /username/repository_name/star
GET     /username/repository_name/stars
DELETE  /username/repository_name/star
```

# 6. High-level design

```bash
            +---------+
            |  Client |
            +----+----+
                 |
                 v
         +---------------+
         | Load Balancer |
         +-------+-------+
                 |
                 v
         +---------------+       +------------------+
         |  API Servers  +------>| Relational DB    |
         +-------+-------+       +------------------+
                 |
                 v
         +---------------+
         | Cache (Redis) |
         +---------------+
                 |
       +------------------------+
       | Returns signed URL for |
       | code blob in S3 via CDN|
       +------------------------+
                 |
                 v
            +------+
            |  CDN |
            +--+---+
               |
               v
             +---+
             |S3 |
             +---+


# Upload path
Client -> LB -> API -> S3

# Download path
Client -> CDN -> S3 (if cache miss)
```

### Key points to remember

- Client: Web browser/Mobile frontend/CLI(git)
- Load Balancer: Can be a layer 7 load balancer (e.g. ALB, NGINX)
- API Servers: Stateless. Handles auth, business logic and serving metadata from the relational DB. Can be scaled horizontally.
- Relational Database: PostgreSQL/MySQL. Might need replicas, backups, possible indexing and sharding.
- Object Storage: S3/GCS. Stores code blobs. Might need lifecycle management, encryption, presigned URLs for downloads.
- CDNs: Fronts S3. Low latency fetches. Clients hit the CDN (via the presigned URL), not the backend server.
- Caching Layer: Redis/Memcached. Might use for hot metadata such as User profiles, repo details, star counts.
- Async Processing: Sending emails/notifications.
- Monitoring

# 7. SQL queries

### User profiles

```sql
-- # GET /user/username/follower-count

select :username, count(*) as follower_cnt
from Followers
where following_id = (
        select id from Users where username = :username limit 1
)

-------------------------------------------

-- # GET /user/username/followers

select u.username
from Followers f
join Users u
on f.follower_id = u.id
where following_id = (
        select id from Users where username = :username limit 1
) order by f.created_at desc
limit 10 offset 0

-------------------------------------------

-- # GET /user/username/following-count

select :username, count(*) as following_cnt
from Followers
where follower_id = (
        select id from Users where username = :username limit 1
)

-------------------------------------------

-- # GET /user/username/following

select u.username
from Followers f
join Users u
on f.following_id = u.id
where follower_id = (
        select id from Users where username = :username limit 1
) order by f.created_at desc
limit 10 offset 0

```

### Repository

```sql
-- # GET /user/username/repositories

-- Approach 1. Using joins
select *
from Repository r
left join Users u
on r.owner_id = u.id
where u.username = :username
order by r.updated_at desc
limit 10 offset 10

-- Approach 2. Using subquery
with cte as (
        select id, username
        from Users
        where username = :username
        limit 1
)

select *, cte.username as owner
from Repository r
join cte cte
on r.owner_id = cte.id
order by updated_at desc
limit 10 offset 0

-------------------------------------------

-- # GET /repositories/search?q=...

select *
from Repository
where name ilike '%...%'
```

### Fork

```sql
-- # DELETE /username/repository_name/fork
delete from Repository
where owner_id = (
    select id from Users where username = :username limit 1
)
and name = :repository_name
and forked_from_repo_id is not null

-------------------------------------------

-- # GET /username/repository_name/forks

with cte as (
        select id
        from Repository
        where owner_id = (
                select id from Users
                where username = :username limit 1
        ) and name = :repository_name
)
select *
from Repository r
join cte cte
on r.forked_from_repo_id = cte.id
```

### Star

```sql

-- # GET /username/repository_name/stars

with cte as (
        select id, owner_id, name
        from Repository
        where owner_id = (
                select id from Users where username = :username limit 1
        ) and name = :repository_name
)

-- Get the number of stars

select cte.name as repository_name, count(*) as starred_count
from Stars s
join cte cte
on s.repository_id = cte.id

-- Get the list of users who starred a repository

select u.username as username
from Stars s
join cte cte
on s.repository_id = cte.id
join Users u
on s.user_id = u.id
order by s.created_at desc
limit 10 offset 0

-------------------------------------------

-- # DELETE /username/repository_name/star
-- An authenticated user USER is removing a star

delete from Stars
where repository_id = (
        select id
        from Repository
        where owner_id = (
                select id from Users where username = :username limit 1
        )
) and user_id = (
        select id from Users where username = :USER limit 1
)

```
