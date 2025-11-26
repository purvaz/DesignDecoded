# YouTube

YouTube is a regular video-sharing platform that allows users to upload, view, and interact with video content. 

## 1. Requirements: 
For any regular streaming platform like YouTube, there are some functional and non-functional requirements we need to consider. 

#### Functional Requirements
1. Users should be able to upload videos
2. Users should be able to watch/stream videos

#### Nonfunctional Requirements
1. Availability >>> Consistency in video uploads
2. Support uploading and streaming for large videos
3. Low latency streaming <500ms even in low bandwidth
4. Scalability to scale to 1M uploads/day and 100M views

#### Assumptions:
1. ~1M uploads 
2. 100M Daily active users
3. Max video size of 256GB or 12 Hours

## 2. Core Entities

- Videos
- Users
- VideoMetaData

## 3. APIs or Interfaces
For this, it is pretty straightforward when you have the functional requirements in place. Just go to the functional requirements and one by one, create the APIs to satisfy them. 

**1. Uploading a video**
POST /videos
{
	Video,
	VideoMetadata
}
This is a POST request (obviously) since we are uploading the data to YouTube. This, however, will only work for smaller videos and not for the larger ones due to the restrictions on APIs for the amount of data transferred. 

**2. Watch a Video**
GET /videos/:videoId -> 
returns {
	Video,
	VideoMetadata
}

## 4. High-Level Design
HLD satisfies the Functional Requirements.

#### Basic Structure
- The client or the user sends a request to the API gateway. 
- The API gateway performs the usual functions like routing the request to the appropriate microservice and also acts as a middleware. 
- The API gateway then sends the request to a Video Service. 
- The video service is connected to two of the databases. One is for storing the Video Metadata (Dynamo / PostgreSQL) and the Videos are stored in BLOB storage (S3, GCS).

#### Uploading Content
- Now, the most glaring issue here is that most API gateways have some limit on the upload size or the amount of data that is being sent via the request. For AWS, it is 10mb and this doesn’t cut with the max size that we are supporting. 
For this we can go via the manual route, i.e. uploading the video in chunks and then manually stitching it up in the video service and then storing it in the Video DB. 
- Otherwise, another option is directly uploading it to the BLOB storage instead of going via the API gateway. Here, we bypass the whole part by part uploading and stitching of the video. S3 supports multipart upload and similarly, GCS supports resumable uploads. With S3, it allows uploading the videos in parts and then S3 automatically stitches the video up.

#### Uploading with Presigned URLs
- Now, instead of sending the video itself via the API gateway, we will be sending the videoMetaData and storing it in the DB. And then using the size sent, the video service will get pre-signed URLs from the video DB (S3) for uploading the video directly. 
- For example, for a video of size 1GB, the videoMetaData will carry the size, and the video service will use it to get the pre-signed URLs from S3. Suppose S3 returns around 100 pre-signed URLs for a video of size 1GB, it will be returned to the user’s end. 
- Now, at the user’s end, the video will be broken down into chunks, using the multipart upload API and then will perform PUT using the */presignedurls* and upload it to S3. 
- Now, in the videoMetaData DB, there will be certain data that will be stored for the Videos, like id, title, description, status and the s3url where the actual video is stored. Initially, the status will be pending until the video is uploaded directly to S3. 
- Once the video is uploaded, S3 will notify the user and user will trigger a status update to the videoMetaData DB. However, we don't wanna rely on the user for this. The best practice would be updating the videoMetaData directly via the S3 storage. 
- S3 actually has a function for this where the notifications can directly be sent to the desired DB once some function is completed. We can set up trigger notifications for the upload complete status and it will be triggered and the notification will be sent to the videoMetaData DB once upload is complete.

#### Downloading / Watching Videos
- Similarly, for watching the videos, the user will make the request for fetching the video to the video service, which will get metadata info from the videoMetaData DB. videoMetaData will return the metadata along with s3url and the user will directly request the S3 database for the video. **This is optimized for large videos in Deep Dive #1.**

## 5. Deep Dives
Deep Dives satisfy the Non-Functional Requirements. 

#### 1. Support uploading and streaming for large videos: \
**Problem:** The current design has some key limitations. For example, if the video is 10GB large and if the user wants to upload or download the video, we need to consider a few things. The user will need to have 10GB of free memory, stable connection and also, the user will need to wait till the video is downloaded. We do not want the user to wait for a long time. Streaming should start immediately. \
**Solution:** In order to solve this issue, we will be adding a new service called “chunker” which will simply divide the video into chunks. When a video is uploaded to the S3 database, a S3 notification is sent to the chunker in addition to the videoMetaData. The chunker will break the video into smaller chunks of 2-10 second clips and return the set of chunks back to S3. It will also send the set of ordered urls to videoMetaData. videoMetaData will store the ordered list of the s3urls and return this whenever the user needs to download the video instead of sending the full s3url of the video. \
**Problem:** A question might arise that why are we performing redundant operation of breaking the video down into chunks before uploading and then again stitching it up at S3 only to break it into chunks again. \
**Solution:** The reality is that the two chunks are optimized it for the function that is being performed. The upload may have larger chunks to minimize the number of requests. Whereas for downloading, the videos are chunked into smaller parts to optimize the video playback experience. 

#### 2. Low latency streaming <500ms even in low bandwidth:\
**Problem:** Chunking satisfies the low latency streaming, but we need to handle the low bandwidth support requirement. Even though we have chunks, it might take the chunk of 2 seconds maybe 10 mins to download with a bad internet connection if the video is 4k.\
**Solution:** For this, we can have a transcoder, which will take a video of the resolution that it has been uploaded at (highest resolution) and make copies with lower resolution to support lesser bandwidth users. When a video is chunked, the set of chunks can be sent to the transcoders. There can be different transcoders for different resolutions. Like one for 4k, one for 1080p, one for 720p and so on until 240p. All of these work in parallel and the resulting data is sent to videoMetaData to store the chunks of different resolutions. Now, when the user requests for a particular video, the appropriate resolution chunks of urls can be sent to the user based on the network connectivity at the user’s end. \
**Problem:** What about when the network connectivity changes mid video? What if the user is watching a video at a stable connection at home, and then decided to leave their house and go out where the connection has been downgraded to 3G? \
**Solution:** We need to have adaptive bit rate for this. Instead of grabbing all the chunks of the same resolution, a periodic check is performed at the user’s end to see if the network is stable. If the connection bandwidth changes, then the appropriate chunks are retrieved from the database. \

#### 3.Availability >>> Consistency in video uploads:
We have done this inherently by doing async processing of the video using chunks.

#### 4. Scalability to scale to 1M uploads/day and 100M views:
The video service is stateless so it can scale horizontally as many times as needed. The API gateway can act as a load balancer to balance appropriately. If the API gateway does not inherently support load balancing, we can add a load balancer in between. S3 is also “infinitely scalable” so there is not limit of storage. For the videoMetaData DB, we dont need much space for all the metadata info. At the most it will need \
<p align="center"> 1Million * 1kb * 365 days = 0.35TB  </p> \
per year, which can easily be stored in a single shard of the database. But if it does exceed the capacity, we can implement sharding by videoId and sort by the video date. We can scale the chunking service and transcoders as needed. 











