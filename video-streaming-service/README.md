# Designing a Video Streaming Platform 📹

## System Requirements 

#### Write Assumptions
- Daily uploads: `10K`
- Resolutions we want to store: `1080p, 720p, 480p and 360p`
- The average duration of videos is `15 mins`
- Average size of each video is `500 MB`
We are dealing with  `10K * 500 MB * 4 = 20 TB` of video content uploaded daily 👀.

#### Read Assumptions
- Average active users at any given moment: `1M`
- Amount of data consumed by user in a second: `500 / 15 MB/min or 568.88 KB/sec`
That's about `1M * 568.88 KB/sec or 568 GB/Sec` 🤯

#### Functional requirements
- High availability 🌈
- Low latency ⚡️
- Consistency 🔥
- Videos available in various resolutions (1080p, 720p, 480p and 360p) 📹

## Overview 🗺
![Overview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eacskn46l6mfolz19whl.png)

### Services required
For the scope of this blog, we'll focus on these major services and components. 
1. **Upload Service** - Upload service for users to upload videos. It should also take care of encoding, compressing, and storing the videos in different resolutions in the blob storage. 
2. **Streaming Service** - Read service for users to consume videos. 
3. **Search service** - Search service for searching videos and channels.
4. **Database** - Considering the traffic and scale of the service we can go with a non-relational database([MongoDB](https://www.mongodb.com), [Apache Cassandra](https://cassandra.apache.org/_/index.html), etc..) or a relational database([PostgreSQL](https://www.postgresql.org/), [MySQL](https://www.mysql.com/), etc...) and [Redis](https://redis.io) for in-memory caching.

#### Data that we'll be storing:
- Video content - Blob storage
- Video metadata (title, description, thumbnail, etc... ) - Relational database
- User metadata - Relational database
- Activity logs and analytics

## DB Schema
![DB Schema](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yk6kbu3o3hi1i1bhx4rq.png)


## Upload Service 📹
![Upload Service](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/pyxvk68klmjpx257zq0j.png)
#### Load Balancer
A load balancer is responsible for distributing traffic across available servers. It acts as a reverse proxy between the client and our application. As we have distributed our system into a microservice architecture where each service is responsible for a particular action, a load balancer will help us scale. 
#### Uploading
Uploading video metadata can be done over a single REST call but for video files, it isn't possible rather we can use a standard streaming protocol to upload video files from the client to our system. [Dynamic Adaptive Streaming over HTTP (DASH)](https://developer.mozilla.org/en-US/docs/Web/Media/DASH_Adaptive_Streaming_for_HTML_5_Video), [Apple HTTP Live Streaming (HLS)](https://developer.apple.com/streaming/), [Adobe HTTP Dynamic Streaming](https://business.adobe.com/in/products/primetime/adobe-media-server/hds-dynamic-streaming.html), or [Microsoft Smooth Streaming](https://learn.microsoft.com/en-us/iis/media/smooth-streaming/smooth-streaming-transport-protocol#:~:text=IIS%20Smooth%20Streaming,%20part%20of,demand%20content%20and%20live%20events.) are some commonly used streaming protocols.
For extremely large video files, we may compress them before passing them into the queue.

#### Processing Queue ⚙️

![Processing Queue](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/h80qpz784qu4q600g6re.png)
Regardless of the quality of the video uploaded by the user, we need to process and save it in multiple resolutions which is a very resource-intensive process and we'll be using a parallel process queue. Each queue takes care of the encoding process and converts the video into a different resolution and uploads it to the blob storage. 

#### Blob storage 🗄
A [Binary Large Object (BLOB)](https://developer.mozilla.org/en-US/docs/Web/API/Blob) is a collection of binary data stored as a single entity in a database management system. This can be used to store encoded videos generated from the process queues. [AWS S3](https://aws.amazon.com/s3/?did=ft_card&trk=ft_card), [Azure Blob Storage](https://azure.microsoft.com/en-in/products/storage/blobs/), and [Google Cloud Storage](https://cloud.google.com/storage) are a few available services that we can use.
To further optimize this solution we can move the most popular videos to a CDN cache.

## Streaming 📺 & Search 🔎 Service 
![Streaming Service](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/nq540up2z2zk6zba665p.png)

#### Elasticsearch

[Elasticsearch](https://www.elastic.co/) is a fast and scalable solution for searching. It consists of documents which are the basic units of information. It may consist of plain text or it can also be a structured JSON. A collection of these documents that have similar characteristics make up an index. It’s an entity that can be queried against in elasticsearch.

#### Logstash

[Logstash](https://www.elastic.co/logstash/) is an important component that is often used with elasticsearch to aggregate and process data. It acts as a data pipeline between your database and the elasticsearch service.

#### Streaming Service

The streaming service will query for the requested video metadata but the actual video file will be streamed from our CDN via the same protocol that we used in our upload service. To optimize the video streaming service we can add an extra layer of cache where we’ll store the metadata(id, title, description, cdn_url, etc) to further optimize the response time.

#### CDN

A Content Delivery Network can be used for caching the videos on a group of geographically distributed but connected servers for more efficient serving to users. The benefits of using a CDN can be:

-   **Efficiency.** CDNs improve webpage loading times and reduce bounce rates. Both advantages keep users from abandoning a slow-loading site or e-commerce application.
-   **Security.** CDNs enhance security with services such as DDoS mitigation, WAFs, and bot mitigation.
-   **Availability.** CDNs can handle more traffic and avoid network failures better than the origin server, increasing content availability.
-   **Optimization.** These networks provide a diverse mix of performance and web content optimization services that complement cached site content.
-   **Resource and cost savings.** CDNs reduce bandwidth consumption and costs.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/av8w6z1kuj52jzlas5c8.png)


When a client requests a video, first we look for it in the CDN cache, if it’s present in the CDN then we can directly stream it from the CDN via DASH, or look else we can stream it from our blob storage and cache it to out CDN as a background process.
-   A few popular options of CDNs for video streaming services are
-   [Netflix Open Connect](https://openconnect.netflix.com/en/)
-   [Cloudflare Stream](https://www.cloudflare.com/products/cloudflare-stream/)
-   [BaishanCloud](https://intl.baishancloud.com/)
-   [AWS CloudFront](https://aws.amazon.com/cloudfront/media/)

## Other Services 📈
Analytics and logging services are necessary for an end-to-end overview of our whole system. They help identify bottlenecks and scale our micro-services.  	
![Analytics](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0osrpmkz79bp0xzeybkv.png)
Logs and analytics can help us identify services that are under heavy load and can be scaled accordingly, based on requirements and user patterns that we derive from the logs. It will also be useful if we decide to add a recommendation system later on.
To store logs at such a huge scale we would need a data warehouse. 
#### What is a data warehouse?
A data warehouse is a more structured and sophisticated database. It stores your data for you, yes, but it also provides context, history, analysis, organization, and possibly even AI parsing.
These extra features make data warehouses an effective way to store vast quantities of data. And by vast, we're talking about data pools that go beyond terabytes. Businesses collect petabytes of data from the apps, communications, and services their teams and customers are using.
1. [Snowflake](https://www.snowflake.com/en/)
2. [Google BigQuery](https://cloud.google.com/bigquery)
3. [Amazon Redshift](https://aws.amazon.com/redshift/)
4. [Azure Synapse Analytics](https://azure.microsoft.com/en-in/products/synapse-analytics)
5. [IBM Db2 Warehouse](https://www.ibm.com/in-en/products/db2/warehouse)
6. [Firebolt](https://www.firebolt.io/)