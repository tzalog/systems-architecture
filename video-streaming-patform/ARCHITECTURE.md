# PeerTube Streaming Platform Architecture on AWS

This document outlines the high-level architecture for the PeerTube-based streaming platform deployed on AWS, optimized for global performance and scalability using a CDN-first approach.

## Architecture Diagram Overview

The system is designed to handle high-concurrency video streaming by offloading static and video content delivery to edge locations.

### Key Components

1.  **Route 53**: AWS's highly available and scalable Domain Name System (DNS) web service. It routes user requests for the `www` domain to the CloudFront distribution.
2.  **CloudFront (CDN)**: The Content Delivery Network caches video files and static assets (thumbnails, JS, CSS) at edge locations. This ensures low latency and high-speed delivery for users worldwide while reducing the load on the backend application servers. It is configured to route dynamic application traffic directly to the Load Balancer.
3.  **S3 Bucket**: Serves as the primary object storage for all video files, thumbnails, and static assets. PeerTube is configured to use S3 for storage, allowing CloudFront to pull content directly from the bucket (via Origin Access Control/OAC).
4.  **Application Load Balancer (ALB)**: Acts as the entry point for all dynamic traffic. It performs health checks on the PeerTube application tier and distributes incoming requests across multiple EC2 instances.
5.  **EC2 Auto Scaling Group (ASG)**: Hosts the PeerTube application tier. This layer runs Nginx as a reverse proxy, the Node.js application logic, and the video transcoding engines. The ASG ensures that instances are added or removed based on CPU/Memory utilization.
6.  **RDS (PostgreSQL)**: A managed relational database service used to store all of PeerTube's metadata, including user profiles, video metadata, comments, and federation data.
7.  **ElastiCache (Redis)**: A managed in-memory data store used by PeerTube for session management, internal caching, and as a message broker for the background job queue (e.g., federation tasks and video processing notifications).

## Performance and Experience Optimization

- **CDN Caching**: By utilizing CloudFront, we minimize the distance between the data and the user, significantly reducing "time to first byte" (TTFB) and buffering.
- **Offloaded Storage**: Using S3 for video storage removes the disk I/O bottleneck from the application servers, allowing them to focus on processing requests and transcoding.
- **Scalability**: The use of an Auto Scaling Group and RDS allows the platform to grow horizontally as the user base and video library expand.
- **Federation Ready**: The architecture supports PeerTube's ActivityPub-based federation, ensuring that incoming and outgoing requests from other instances are handled efficiently.
