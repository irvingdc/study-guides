# System Design Study Guide - Part 7: Real-World Case Studies

## 1. Design a URL Shortener (bit.ly)

### Requirements Analysis

**Functional Requirements:**
- Shorten long URLs to 7-character codes
- Redirect short URLs to original URLs
- Custom aliases (optional)
- Analytics (click count, referrers)
- URL expiration (optional)

**Non-Functional Requirements:**
- 100M URLs shortened per month
- 10:1 read/write ratio (1B reads per month)
- < 100ms latency
- 99.9% availability
- URLs stored for 5 years by default

### Capacity Estimation

```python
class URLShortenerCapacity:
    def calculate_requirements(self):
        # Traffic estimates
        writes_per_month = 100_000_000
        reads_per_month = writes_per_month * 10
        
        # QPS calculations
        writes_per_second = writes_per_month / (30 * 24 * 3600)  # ~40 writes/sec
        reads_per_second = reads_per_month / (30 * 24 * 3600)    # ~400 reads/sec
        
        # Storage estimates
        avg_url_size = 500  # bytes (including metadata)
        storage_per_year = writes_per_month * 12 * avg_url_size
        total_storage_5_years = storage_per_year * 5 / (1024**4)  # ~3 TB
        
        # Bandwidth estimates
        write_bandwidth = writes_per_second * avg_url_size  # ~20 KB/s
        read_bandwidth = reads_per_second * avg_url_size    # ~200 KB/s
        
        # Cache requirements (80/20 rule)
        hot_urls = writes_per_month * 0.2
        cache_size = hot_urls * avg_url_size / (1024**3)  # ~10 GB
        
        return {
            'writes_qps': writes_per_second,
            'reads_qps': reads_per_second,
            'storage_tb': total_storage_5_years,
            'cache_gb': cache_size,
            'bandwidth_kbps': (write_bandwidth + read_bandwidth) / 1024
        }
```

### System Architecture

```python
class URLShortenerArchitecture:
    """
    Components:
    - Load Balancer (AWS ALB)
    - Web Servers (Stateless)
    - Cache Layer (Redis)
    - Database (DynamoDB/Cassandra)
    - Analytics Service (Kafka + ClickHouse)
    - CDN for redirects
    """
    
    def __init__(self):
        self.base_url = "https://short.ly/"
        self.url_length = 7
        self.charset = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
```

### Database Design

```sql
-- NoSQL Schema (DynamoDB)
{
    "short_url": "abc123d",  -- Partition Key
    "long_url": "https://example.com/very/long/url",
    "created_at": 1704067200,
    "expires_at": 1861747200,  -- Optional
    "user_id": "user_123",      -- Optional
    "custom_alias": false,
    "click_count": 0
}

-- Analytics Table
{
    "short_url": "abc123d",     -- Partition Key
    "timestamp": 1704067200,    -- Sort Key
    "ip_address": "192.168.1.1",
    "user_agent": "Mozilla/5.0",
    "referrer": "https://google.com",
    "country": "US"
}
```

### Core Implementation

```python
import hashlib
import base62
import time
from datetime import datetime, timedelta

class URLShortener:
    def __init__(self, cache, database, counter_service):
        self.cache = cache
        self.db = database
        self.counter = counter_service
        
    def shorten_url(self, long_url, custom_alias=None, user_id=None):
        """Shorten a URL using counter-based approach"""
        
        # Check if custom alias is requested
        if custom_alias:
            if self.is_alias_available(custom_alias):
                short_url = custom_alias
            else:
                raise ValueError("Custom alias already taken")
        else:
            # Generate short URL using distributed counter
            counter_value = self.counter.get_next_id()
            short_url = self.encode_base62(counter_value)
        
        # Store in database
        url_record = {
            'short_url': short_url,
            'long_url': long_url,
            'created_at': int(time.time()),
            'user_id': user_id,
            'custom_alias': custom_alias is not None,
            'click_count': 0
        }
        
        self.db.put_item(url_record)
        
        # Cache popular URLs
        self.cache.set(short_url, long_url, ttl=3600)
        
        return f"{self.base_url}{short_url}"
    
    def resolve_url(self, short_url):
        """Resolve short URL to long URL"""
        
        # Check cache first
        long_url = self.cache.get(short_url)
        
        if long_url:
            # Async increment counter
            self.async_increment_counter(short_url)
            return long_url
        
        # Cache miss - check database
        url_record = self.db.get_item(short_url)
        
        if not url_record:
            raise ValueError("URL not found")
        
        # Check expiration
        if 'expires_at' in url_record:
            if url_record['expires_at'] < time.time():
                raise ValueError("URL has expired")
        
        long_url = url_record['long_url']
        
        # Update cache
        self.cache.set(short_url, long_url, ttl=3600)
        
        # Async update analytics
        self.async_track_click(short_url)
        
        return long_url
    
    def encode_base62(self, num):
        """Convert number to base62 string"""
        charset = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        result = []
        
        while num > 0:
            result.append(charset[num % 62])
            num //= 62
        
        # Pad to minimum length
        while len(result) < 7:
            result.append('0')
        
        return ''.join(reversed(result))

class DistributedCounter:
    """Distributed counter for generating unique IDs"""
    
    def __init__(self, worker_id, redis_client):
        self.worker_id = worker_id
        self.redis = redis_client
        self.local_counter = 0
        self.range_start = 0
        self.range_end = 0
        
    def get_next_id(self):
        """Get next unique ID"""
        
        if self.local_counter >= self.range_end:
            # Need new range
            self.allocate_new_range()
        
        current_id = self.local_counter
        self.local_counter += 1
        
        return current_id
    
    def allocate_new_range(self):
        """Allocate new ID range from Redis"""
        
        # Atomic increment by 1000
        new_end = self.redis.incrby('url_counter', 1000)
        
        self.range_start = new_end - 1000
        self.range_end = new_end
        self.local_counter = self.range_start
```

### Scaling Considerations

```python
class URLShortenerScaling:
    def implement_caching_strategy(self):
        """Multi-level caching strategy"""
        
        return {
            'cdn_cache': {
                'provider': 'CloudFlare',
                'ttl': 86400,  # 1 day
                'cache_key': 'short_url',
                'purpose': 'Cache redirects at edge'
            },
            'application_cache': {
                'technology': 'Redis Cluster',
                'ttl': 3600,  # 1 hour
                'eviction': 'LRU',
                'size': '10GB',
                'purpose': 'Cache hot URLs'
            },
            'database_cache': {
                'technology': 'DynamoDB DAX',
                'ttl': 300,  # 5 minutes
                'purpose': 'Cache database queries'
            }
        }
    
    def implement_rate_limiting(self):
        """Rate limiting to prevent abuse"""
        
        return {
            'per_user': {
                'create_url': '100 per hour',
                'resolve_url': '10000 per hour'
            },
            'per_ip': {
                'create_url': '50 per hour',
                'resolve_url': '5000 per hour'
            },
            'implementation': 'Redis sliding window'
        }
```

## 2. Design a Social Media Feed (Twitter Timeline)

### Requirements

**Functional Requirements:**
- Users can post tweets (280 characters)
- Users can follow other users
- Home timeline shows tweets from followed users
- User can like, retweet, reply
- Search tweets
- Trending topics

**Non-Functional Requirements:**
- 500M users, 100M DAU
- Average user follows 200 people
- Average user posts 2 tweets/day
- Timeline generation < 500ms
- Eventually consistent is acceptable

### Architecture Design

```python
class TwitterArchitecture:
    """
    Key Components:
    - Tweet Service
    - Timeline Service
    - User Graph Service
    - Media Service
    - Search Service
    - Notification Service
    """
    
    def __init__(self):
        self.fanout_strategy = 'hybrid'  # push, pull, or hybrid
```

### Database Schema

```sql
-- User Table (PostgreSQL)
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    follower_count INT DEFAULT 0,
    following_count INT DEFAULT 0,
    is_celebrity BOOLEAN DEFAULT FALSE  -- For hybrid approach
);

-- Tweet Table (Cassandra for scalability)
CREATE TABLE tweets (
    tweet_id UUID,
    user_id BIGINT,
    content TEXT,
    created_at TIMESTAMP,
    likes_count INT,
    retweets_count INT,
    replies_count INT,
    media_urls LIST<TEXT>,
    PRIMARY KEY (user_id, created_at, tweet_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Follow Relationships (Graph Database or Cassandra)
CREATE TABLE follows (
    follower_id BIGINT,
    following_id BIGINT,
    created_at TIMESTAMP,
    PRIMARY KEY (follower_id, following_id)
);

-- Timeline Cache (Redis)
-- Key: timeline:user_id
-- Value: List of tweet IDs with scores (timestamp)
```

### Timeline Generation Strategies

```python
class TimelineService:
    def __init__(self):
        self.celebrity_threshold = 10000  # followers
        
    def generate_timeline_hybrid(self, user_id):
        """
        Hybrid approach:
        - Push for regular users
        - Pull for celebrities
        """
        
        # Get user's following list
        following = self.get_following(user_id)
        
        celebrities = []
        regular_users = []
        
        for followed_user in following:
            if followed_user['follower_count'] > self.celebrity_threshold:
                celebrities.append(followed_user['user_id'])
            else:
                regular_users.append(followed_user['user_id'])
        
        timeline = []
        
        # Pull from celebrities (real-time)
        for celebrity_id in celebrities:
            recent_tweets = self.get_recent_tweets(celebrity_id, limit=10)
            timeline.extend(recent_tweets)
        
        # Get pre-computed timeline for regular users
        cached_timeline = self.get_cached_timeline(user_id)
        timeline.extend(cached_timeline)
        
        # Sort by timestamp
        timeline.sort(key=lambda x: x['created_at'], reverse=True)
        
        # Apply ranking algorithm
        ranked_timeline = self.rank_timeline(timeline, user_id)
        
        return ranked_timeline[:100]  # Return top 100 tweets
    
    def fanout_on_write(self, tweet):
        """Push model - write to followers' timelines"""
        
        user_id = tweet['user_id']
        followers = self.get_followers(user_id)
        
        # Check if celebrity
        if len(followers) > self.celebrity_threshold:
            # Don't fanout for celebrities
            return
        
        # Fanout to all followers
        for follower_id in followers:
            self.add_to_timeline(follower_id, tweet)
    
    def add_to_timeline(self, user_id, tweet):
        """Add tweet to user's cached timeline"""
        
        timeline_key = f"timeline:{user_id}"
        
        # Add to Redis sorted set
        self.redis.zadd(
            timeline_key,
            {tweet['tweet_id']: tweet['created_at']}
        )
        
        # Trim timeline to last 1000 tweets
        self.redis.zremrangebyrank(timeline_key, 0, -1001)
        
        # Set expiration
        self.redis.expire(timeline_key, 86400)  # 1 day
    
    def rank_timeline(self, tweets, user_id):
        """Rank tweets using ML model"""
        
        user_features = self.get_user_features(user_id)
        
        ranked_tweets = []
        for tweet in tweets:
            score = self.calculate_relevance_score(tweet, user_features)
            ranked_tweets.append({
                'tweet': tweet,
                'score': score
            })
        
        # Sort by score
        ranked_tweets.sort(key=lambda x: x['score'], reverse=True)
        
        return [item['tweet'] for item in ranked_tweets]
    
    def calculate_relevance_score(self, tweet, user_features):
        """Calculate tweet relevance score"""
        
        score = 0
        
        # Recency
        age_hours = (time.time() - tweet['created_at']) / 3600
        recency_score = 1 / (1 + age_hours / 24)
        score += recency_score * 0.3
        
        # Engagement
        engagement_score = (
            tweet['likes_count'] * 0.1 +
            tweet['retweets_count'] * 0.3 +
            tweet['replies_count'] * 0.2
        ) / 1000
        score += min(engagement_score, 1) * 0.4
        
        # User affinity
        affinity_score = user_features.get(tweet['user_id'], 0)
        score += affinity_score * 0.3
        
        return score
```

### Tweet Posting Flow

```python
class TweetService:
    def post_tweet(self, user_id, content, media=None):
        """Post a new tweet"""
        
        # Validate tweet
        if len(content) > 280:
            raise ValueError("Tweet too long")
        
        # Generate tweet ID
        tweet_id = str(uuid.uuid4())
        
        # Process media if present
        media_urls = []
        if media:
            media_urls = self.upload_media(media)
        
        # Create tweet object
        tweet = {
            'tweet_id': tweet_id,
            'user_id': user_id,
            'content': content,
            'media_urls': media_urls,
            'created_at': int(time.time()),
            'likes_count': 0,
            'retweets_count': 0,
            'replies_count': 0
        }
        
        # Store tweet
        self.store_tweet(tweet)
        
        # Fanout to followers (async)
        self.async_fanout(tweet)
        
        # Update search index (async)
        self.async_index_tweet(tweet)
        
        # Extract and track hashtags (async)
        self.async_process_hashtags(tweet)
        
        return tweet_id
    
    def async_fanout(self, tweet):
        """Asynchronously fanout tweet to followers"""
        
        # Send to message queue
        self.queue.send('fanout_queue', {
            'action': 'fanout_tweet',
            'tweet': tweet
        })
    
    def process_fanout_task(self, task):
        """Process fanout task from queue"""
        
        tweet = task['tweet']
        user_id = tweet['user_id']
        
        # Get followers
        followers = self.get_followers(user_id)
        
        # Batch process
        batch_size = 1000
        for i in range(0, len(followers), batch_size):
            batch = followers[i:i+batch_size]
            
            # Multi-threaded fanout
            with ThreadPoolExecutor(max_workers=10) as executor:
                futures = []
                for follower_id in batch:
                    future = executor.submit(
                        self.add_to_timeline,
                        follower_id,
                        tweet
                    )
                    futures.append(future)
                
                # Wait for completion
                for future in futures:
                    future.result()
```

## 3. Design a Video Streaming Platform (YouTube/Netflix)

### Requirements

**Functional Requirements:**
- Upload videos
- Stream videos (adaptive bitrate)
- Search videos
- Recommendations
- Comments and likes
- Subtitles support

**Non-Functional Requirements:**
- 1B users, 100M DAU
- 5M videos, average 100MB each
- Global distribution
- < 200ms startup time
- Support 4K streaming

### Video Processing Pipeline

```python
class VideoProcessingPipeline:
    def __init__(self):
        self.supported_resolutions = [
            {'name': '4K', 'width': 3840, 'height': 2160, 'bitrate': 35000},
            {'name': '1080p', 'width': 1920, 'height': 1080, 'bitrate': 8000},
            {'name': '720p', 'width': 1280, 'height': 720, 'bitrate': 5000},
            {'name': '480p', 'width': 854, 'height': 480, 'bitrate': 2500},
            {'name': '360p', 'width': 640, 'height': 360, 'bitrate': 1000}
        ]
    
    def process_upload(self, video_file, metadata):
        """Process uploaded video"""
        
        # Generate video ID
        video_id = str(uuid.uuid4())
        
        # Upload original to S3
        original_url = self.upload_to_s3(video_file, f"originals/{video_id}")
        
        # Create processing job
        job = {
            'video_id': video_id,
            'original_url': original_url,
            'metadata': metadata,
            'status': 'queued',
            'created_at': time.time()
        }
        
        # Queue for processing
        self.queue_processing_job(job)
        
        return video_id
    
    def transcode_video(self, job):
        """Transcode video to multiple resolutions"""
        
        video_id = job['video_id']
        original_url = job['original_url']
        
        # Download original
        video_file = self.download_from_s3(original_url)
        
        transcoded_files = []
        
        for resolution in self.supported_resolutions:
            # Transcode using FFmpeg
            output_file = self.transcode_with_ffmpeg(
                video_file,
                resolution
            )
            
            # Create HLS segments
            segments = self.create_hls_segments(output_file)
            
            # Upload segments to CDN
            segment_urls = self.upload_segments_to_cdn(
                segments,
                video_id,
                resolution['name']
            )
            
            transcoded_files.append({
                'resolution': resolution['name'],
                'manifest_url': segment_urls['manifest'],
                'segments': segment_urls['segments']
            })
        
        # Update video metadata
        self.update_video_metadata(video_id, transcoded_files)
        
        # Generate thumbnails
        thumbnails = self.generate_thumbnails(video_file)
        self.store_thumbnails(video_id, thumbnails)
        
        return transcoded_files
    
    def create_hls_segments(self, video_file):
        """Create HLS segments for adaptive streaming"""
        
        # FFmpeg command for HLS
        command = [
            'ffmpeg',
            '-i', video_file,
            '-c:v', 'h264',
            '-c:a', 'aac',
            '-hls_time', '10',  # 10 second segments
            '-hls_list_size', '0',
            '-f', 'hls',
            'output.m3u8'
        ]
        
        # Execute and return segments
        return self.execute_ffmpeg(command)
```

### Video Streaming Architecture

```python
class VideoStreamingService:
    def __init__(self):
        self.cdn_providers = ['CloudFront', 'Akamai', 'Fastly']
        
    def stream_video(self, video_id, user_location, device_type):
        """Stream video with adaptive bitrate"""
        
        # Get video metadata
        video = self.get_video_metadata(video_id)
        
        # Select best CDN edge
        cdn_edge = self.select_cdn_edge(user_location)
        
        # Determine initial quality
        initial_quality = self.determine_initial_quality(
            device_type,
            self.measure_bandwidth()
        )
        
        # Generate manifest
        manifest = self.generate_adaptive_manifest(
            video,
            cdn_edge,
            initial_quality
        )
        
        # Track view
        self.track_video_view(video_id, user_id)
        
        return {
            'manifest_url': manifest['url'],
            'cdn_edge': cdn_edge,
            'initial_quality': initial_quality
        }
    
    def generate_adaptive_manifest(self, video, cdn_edge, initial_quality):
        """Generate HLS/DASH manifest for adaptive streaming"""
        
        manifest = {
            'version': 3,
            'streams': []
        }
        
        for quality in video['qualities']:
            stream = {
                'bandwidth': quality['bitrate'] * 1000,
                'resolution': f"{quality['width']}x{quality['height']}",
                'codecs': 'avc1.64001f,mp4a.40.2',
                'url': f"{cdn_edge}/videos/{video['id']}/{quality['name']}/playlist.m3u8"
            }
            manifest['streams'].append(stream)
        
        return manifest
    
    def implement_video_caching(self):
        """Multi-tier video caching strategy"""
        
        return {
            'edge_cache': {
                'location': 'CDN PoPs',
                'content': 'Popular videos, recent segments',
                'ttl': '7 days',
                'strategy': 'LFU with aging'
            },
            'regional_cache': {
                'location': 'Regional data centers',
                'content': 'Top 20% videos per region',
                'ttl': '30 days',
                'strategy': 'Predictive caching'
            },
            'origin': {
                'location': 'Central storage (S3)',
                'content': 'All videos',
                'ttl': 'Permanent',
                'strategy': 'Cold storage for old videos'
            }
        }
```

## 4. Design a Ride-Sharing System (Uber/Lyft)

### Requirements

**Functional Requirements:**
- Request rides
- Match riders with drivers
- Real-time location tracking
- Calculate fare
- Payment processing
- Ratings and feedback

**Non-Functional Requirements:**
- 10M users, 1M daily rides
- Match within 15 seconds
- Location updates every 4 seconds
- 99.9% availability

### Location Service and Matching

```python
import geohash
from math import radians, sin, cos, sqrt, atan2

class LocationService:
    def __init__(self):
        self.driver_locations = {}  # driver_id -> location
        self.geohash_index = {}     # geohash -> set of driver_ids
        
    def update_driver_location(self, driver_id, lat, lon):
        """Update driver location and geohash index"""
        
        # Remove from old geohash
        if driver_id in self.driver_locations:
            old_location = self.driver_locations[driver_id]
            old_geohash = geohash.encode(
                old_location['lat'],
                old_location['lon'],
                precision=6
            )
            self.geohash_index[old_geohash].discard(driver_id)
        
        # Update location
        self.driver_locations[driver_id] = {
            'lat': lat,
            'lon': lon,
            'timestamp': time.time(),
            'heading': self.calculate_heading(driver_id, lat, lon)
        }
        
        # Add to new geohash
        new_geohash = geohash.encode(lat, lon, precision=6)
        if new_geohash not in self.geohash_index:
            self.geohash_index[new_geohash] = set()
        self.geohash_index[new_geohash].add(driver_id)
        
        # Store in Redis with TTL
        self.redis.setex(
            f"driver:location:{driver_id}",
            60,  # 60 second TTL
            json.dumps({'lat': lat, 'lon': lon})
        )
    
    def find_nearby_drivers(self, rider_lat, rider_lon, radius_km=5):
        """Find available drivers within radius"""
        
        # Get geohashes to search
        center_geohash = geohash.encode(rider_lat, rider_lon, precision=6)
        neighbor_geohashes = geohash.neighbors(center_geohash)
        search_geohashes = [center_geohash] + neighbor_geohashes
        
        nearby_drivers = []
        
        for gh in search_geohashes:
            if gh in self.geohash_index:
                for driver_id in self.geohash_index[gh]:
                    driver_location = self.driver_locations[driver_id]
                    
                    # Calculate distance
                    distance = self.haversine_distance(
                        rider_lat, rider_lon,
                        driver_location['lat'], driver_location['lon']
                    )
                    
                    if distance <= radius_km:
                        nearby_drivers.append({
                            'driver_id': driver_id,
                            'distance': distance,
                            'location': driver_location,
                            'eta': self.calculate_eta(distance)
                        })
        
        # Sort by distance
        nearby_drivers.sort(key=lambda x: x['distance'])
        
        return nearby_drivers
    
    def haversine_distance(self, lat1, lon1, lat2, lon2):
        """Calculate distance between two points"""
        
        R = 6371  # Earth's radius in km
        
        lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
        
        dlat = lat2 - lat1
        dlon = lon2 - lon1
        
        a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
        c = 2 * atan2(sqrt(a), sqrt(1-a))
        
        return R * c

class RideMatchingService:
    def __init__(self):
        self.location_service = LocationService()
        self.pricing_service = PricingService()
        
    def request_ride(self, rider_id, pickup_location, destination):
        """Process ride request"""
        
        # Generate ride ID
        ride_id = str(uuid.uuid4())
        
        # Find nearby drivers
        nearby_drivers = self.location_service.find_nearby_drivers(
            pickup_location['lat'],
            pickup_location['lon']
        )
        
        if not nearby_drivers:
            return {'status': 'no_drivers_available'}
        
        # Calculate estimated fare
        estimated_fare = self.pricing_service.calculate_fare(
            pickup_location,
            destination
        )
        
        # Create ride request
        ride_request = {
            'ride_id': ride_id,
            'rider_id': rider_id,
            'pickup': pickup_location,
            'destination': destination,
            'estimated_fare': estimated_fare,
            'status': 'matching',
            'created_at': time.time()
        }
        
        # Store ride request
        self.store_ride_request(ride_request)
        
        # Send to matching queue
        self.dispatch_to_drivers(ride_request, nearby_drivers)
        
        return {
            'ride_id': ride_id,
            'status': 'matching',
            'estimated_wait': nearby_drivers[0]['eta'],
            'estimated_fare': estimated_fare
        }
    
    def dispatch_to_drivers(self, ride_request, drivers):
        """Dispatch ride request to drivers"""
        
        # Strategy 1: Broadcast to all nearby drivers
        for driver in drivers[:10]:  # Limit to 10 closest
            self.send_ride_offer(driver['driver_id'], ride_request)
        
        # Set timeout for acceptance
        self.schedule_timeout(ride_request['ride_id'], timeout=15)
    
    def handle_driver_response(self, driver_id, ride_id, accepted):
        """Handle driver's response to ride offer"""
        
        if not accepted:
            return
        
        # Use distributed lock to prevent race condition
        lock_key = f"ride:lock:{ride_id}"
        
        with self.redis_lock(lock_key):
            ride = self.get_ride(ride_id)
            
            if ride['status'] != 'matching':
                # Already matched
                return {'status': 'already_matched'}
            
            # Match successful
            ride['driver_id'] = driver_id
            ride['status'] = 'matched'
            ride['matched_at'] = time.time()
            
            self.update_ride(ride)
            
            # Notify rider
            self.notify_rider(ride['rider_id'], {
                'type': 'driver_matched',
                'driver_id': driver_id,
                'eta': self.calculate_driver_eta(driver_id, ride['pickup'])
            })
            
            # Notify other drivers that ride is taken
            self.cancel_other_offers(ride_id, driver_id)
            
            return {'status': 'matched'}
```

## 5. Design a Payment System (Stripe/PayPal)

### Requirements

**Functional Requirements:**
- Process payments (cards, bank transfers, wallets)
- Handle refunds
- Subscription billing
- Fraud detection
- PCI compliance
- Webhook notifications

**Non-Functional Requirements:**
- 10K transactions per second
- 99.99% availability
- < 2 second transaction time
- Zero data loss
- Globally distributed

### Payment Processing Architecture

```python
class PaymentSystem:
    def __init__(self):
        self.payment_methods = ['card', 'bank', 'wallet']
        self.supported_currencies = ['USD', 'EUR', 'GBP', 'JPY']
        
    def process_payment(self, payment_request):
        """Process a payment transaction"""
        
        # Generate unique transaction ID
        transaction_id = self.generate_transaction_id()
        
        # Validate request
        validation_result = self.validate_payment_request(payment_request)
        if not validation_result['valid']:
            return {
                'status': 'failed',
                'error': validation_result['error']
            }
        
        # Fraud check
        fraud_score = self.check_fraud(payment_request)
        if fraud_score > 0.8:
            return self.handle_suspicious_transaction(transaction_id, payment_request)
        
        # Begin transaction (2PC)
        with self.distributed_transaction(transaction_id):
            # Reserve funds
            reservation = self.reserve_funds(
                payment_request['payment_method'],
                payment_request['amount']
            )
            
            if not reservation['success']:
                raise TransactionError(reservation['error'])
            
            # Record transaction
            self.record_transaction({
                'transaction_id': transaction_id,
                'merchant_id': payment_request['merchant_id'],
                'amount': payment_request['amount'],
                'currency': payment_request['currency'],
                'status': 'processing',
                'created_at': time.time()
            })
            
            # Process with payment gateway
            gateway_result = self.process_with_gateway(
                payment_request,
                reservation['token']
            )
            
            if gateway_result['success']:
                # Capture funds
                self.capture_funds(reservation['token'])
                
                # Update transaction status
                self.update_transaction_status(transaction_id, 'completed')
                
                # Send webhook
                self.send_webhook(payment_request['merchant_id'], {
                    'event': 'payment.completed',
                    'transaction_id': transaction_id
                })
                
                return {
                    'status': 'success',
                    'transaction_id': transaction_id
                }
            else:
                # Release reservation
                self.release_funds(reservation['token'])
                raise TransactionError(gateway_result['error'])
    
    def implement_idempotency(self):
        """Implement idempotency for payment requests"""
        
        class IdempotencyHandler:
            def __init__(self, redis_client):
                self.redis = redis_client
                
            def process_with_idempotency(self, idempotency_key, func, *args):
                # Check if already processed
                result = self.redis.get(f"idempotency:{idempotency_key}")
                
                if result:
                    return json.loads(result)
                
                # Acquire lock
                lock = self.redis.lock(f"lock:{idempotency_key}", timeout=10)
                
                if lock.acquire(blocking=True):
                    try:
                        # Double-check after acquiring lock
                        result = self.redis.get(f"idempotency:{idempotency_key}")
                        if result:
                            return json.loads(result)
                        
                        # Process request
                        result = func(*args)
                        
                        # Store result
                        self.redis.setex(
                            f"idempotency:{idempotency_key}",
                            86400,  # 24 hours
                            json.dumps(result)
                        )
                        
                        return result
                    finally:
                        lock.release()
```

## 6. Design a Real Estate Platform (Zillow Specific)

### Requirements

**Functional Requirements:**
- Property listings (for sale, rent)
- Search with filters (location, price, features)
- Zestimate (property value estimation)
- Virtual tours and images
- Saved searches and alerts
- Agent connectivity

**Non-Functional Requirements:**
- 100M property records
- 50M monthly unique users
- Real-time MLS data sync
- < 500ms search response
- Accurate valuations

### Property Search and Indexing

```python
class ZillowSearchService:
    def __init__(self):
        self.elasticsearch = Elasticsearch()
        
    def index_property(self, property_data):
        """Index property for search"""
        
        # Prepare document
        doc = {
            'property_id': property_data['id'],
            'address': property_data['address'],
            'location': {
                'lat': property_data['latitude'],
                'lon': property_data['longitude']
            },
            'price': property_data['price'],
            'bedrooms': property_data['bedrooms'],
            'bathrooms': property_data['bathrooms'],
            'square_feet': property_data['square_feet'],
            'property_type': property_data['type'],
            'listing_type': property_data['listing_type'],
            'features': property_data['features'],
            'school_district': property_data['school_district'],
            'walk_score': property_data['walk_score'],
            'year_built': property_data['year_built'],
            'last_updated': time.time()
        }
        
        # Index in Elasticsearch
        self.elasticsearch.index(
            index='properties',
            id=property_data['id'],
            body=doc
        )
    
    def search_properties(self, search_criteria):
        """Search properties with complex filters"""
        
        # Build query
        query = {
            'bool': {
                'must': [],
                'filter': []
            }
        }
        
        # Location search
        if 'location' in search_criteria:
            if 'polygon' in search_criteria['location']:
                query['bool']['filter'].append({
                    'geo_polygon': {
                        'location': {
                            'points': search_criteria['location']['polygon']
                        }
                    }
                })
            elif 'radius' in search_criteria['location']:
                query['bool']['filter'].append({
                    'geo_distance': {
                        'distance': f"{search_criteria['location']['radius']}mi",
                        'location': search_criteria['location']['center']
                    }
                })
        
        # Price range
        if 'price_min' in search_criteria or 'price_max' in search_criteria:
            price_range = {}
            if 'price_min' in search_criteria:
                price_range['gte'] = search_criteria['price_min']
            if 'price_max' in search_criteria:
                price_range['lte'] = search_criteria['price_max']
            
            query['bool']['filter'].append({
                'range': {'price': price_range}
            })
        
        # Property features
        if 'features' in search_criteria:
            for feature in search_criteria['features']:
                query['bool']['must'].append({
                    'term': {'features': feature}
                })
        
        # Execute search
        results = self.elasticsearch.search(
            index='properties',
            body={
                'query': query,
                'sort': self.get_sort_criteria(search_criteria),
                'size': search_criteria.get('limit', 50),
                'from': search_criteria.get('offset', 0)
            }
        )
        
        return self.format_search_results(results)

class ZestimateService:
    """Property valuation service"""
    
    def __init__(self):
        self.model = self.load_ml_model()
        
    def calculate_zestimate(self, property_id):
        """Calculate property value estimate"""
        
        # Get property details
        property_data = self.get_property_data(property_id)
        
        # Get comparable properties
        comparables = self.find_comparables(property_data)
        
        # Extract features
        features = self.extract_features(property_data, comparables)
        
        # Predict using ML model
        base_estimate = self.model.predict(features)
        
        # Adjust for recent market trends
        market_adjustment = self.calculate_market_adjustment(property_data['location'])
        
        # Apply confidence interval
        estimate = base_estimate * market_adjustment
        confidence = self.calculate_confidence(property_data, comparables)
        
        return {
            'zestimate': estimate,
            'confidence': confidence,
            'range': {
                'low': estimate * (1 - confidence * 0.1),
                'high': estimate * (1 + confidence * 0.1)
            },
            'comparables': comparables[:5],
            'last_updated': time.time()
        }
    
    def find_comparables(self, property_data):
        """Find comparable properties for valuation"""
        
        # Search criteria for comparables
        search_criteria = {
            'location': {
                'center': {
                    'lat': property_data['latitude'],
                    'lon': property_data['longitude']
                },
                'radius': 0.5  # miles
            },
            'bedrooms': {
                'min': property_data['bedrooms'] - 1,
                'max': property_data['bedrooms'] + 1
            },
            'square_feet': {
                'min': property_data['square_feet'] * 0.8,
                'max': property_data['square_feet'] * 1.2
            },
            'sold_date': {
                'min': time.time() - (180 * 86400)  # Last 6 months
            }
        }
        
        # Find similar properties
        comparables = self.search_sold_properties(search_criteria)
        
        # Score comparables
        scored_comparables = []
        for comp in comparables:
            score = self.calculate_comparability_score(property_data, comp)
            scored_comparables.append({
                'property': comp,
                'score': score
            })
        
        # Sort by score
        scored_comparables.sort(key=lambda x: x['score'], reverse=True)
        
        return [c['property'] for c in scored_comparables[:20]]
```

## 7. Design a Collaborative Editing System (Google Docs)

### Requirements

**Functional Requirements:**
- Real-time collaborative editing
- Conflict resolution
- Version history
- Comments and suggestions
- Offline support
- Access control

**Non-Functional Requirements:**
- Support 100 concurrent editors
- < 100ms latency for operations
- No data loss
- Eventual consistency

### Operational Transformation Implementation

```python
class OperationalTransform:
    """
    Implement OT for collaborative editing
    """
    
    def __init__(self):
        self.document_state = ""
        self.version = 0
        self.pending_operations = []
        
    def transform(self, op1, op2):
        """
        Transform op1 against op2
        Both operations happened concurrently
        """
        
        if op1['type'] == 'insert' and op2['type'] == 'insert':
            if op1['position'] < op2['position']:
                return op1, {
                    'type': 'insert',
                    'position': op2['position'] + len(op1['text']),
                    'text': op2['text']
                }
            else:
                return {
                    'type': 'insert',
                    'position': op1['position'] + len(op2['text']),
                    'text': op1['text']
                }, op2
                
        elif op1['type'] == 'delete' and op2['type'] == 'delete':
            if op1['position'] < op2['position']:
                return op1, {
                    'type': 'delete',
                    'position': op2['position'] - op1['length'],
                    'length': op2['length']
                }
            else:
                return {
                    'type': 'delete',
                    'position': op1['position'] - op2['length'],
                    'length': op1['length']
                }, op2
                
        elif op1['type'] == 'insert' and op2['type'] == 'delete':
            if op1['position'] <= op2['position']:
                return op1, {
                    'type': 'delete',
                    'position': op2['position'] + len(op1['text']),
                    'length': op2['length']
                }
            else:
                return {
                    'type': 'insert',
                    'position': op1['position'] - op2['length'],
                    'text': op1['text']
                }, op2
                
        # Handle other cases...
        return op1, op2

class CollaborativeDocument:
    def __init__(self, doc_id):
        self.doc_id = doc_id
        self.content = ""
        self.version = 0
        self.clients = {}  # client_id -> WebSocket
        self.operation_history = []
        
    def handle_client_operation(self, client_id, operation):
        """Handle operation from client"""
        
        # Add version to operation
        operation['version'] = self.version
        operation['client_id'] = client_id
        
        # Apply operation to document
        self.apply_operation(operation)
        
        # Increment version
        self.version += 1
        
        # Store in history
        self.operation_history.append(operation)
        
        # Broadcast to other clients
        self.broadcast_operation(operation, exclude=client_id)
        
        return {'status': 'applied', 'version': self.version}
    
    def apply_operation(self, operation):
        """Apply operation to document"""
        
        if operation['type'] == 'insert':
            pos = operation['position']
            text = operation['text']
            self.content = self.content[:pos] + text + self.content[pos:]
            
        elif operation['type'] == 'delete':
            pos = operation['position']
            length = operation['length']
            self.content = self.content[:pos] + self.content[pos + length:]
            
    def broadcast_operation(self, operation, exclude=None):
        """Broadcast operation to all connected clients"""
        
        message = {
            'type': 'operation',
            'operation': operation,
            'version': self.version
        }
        
        for client_id, websocket in self.clients.items():
            if client_id != exclude:
                websocket.send(json.dumps(message))
    
    def handle_client_sync(self, client_id, client_version):
        """Sync client with latest document state"""
        
        if client_version == self.version:
            return {'status': 'up_to_date'}
        
        # Get operations since client version
        missing_ops = self.operation_history[client_version:]
        
        return {
            'status': 'behind',
            'operations': missing_ops,
            'current_version': self.version
        }
```

## 8. Design a Distributed File Storage (Dropbox)

### Requirements

**Functional Requirements:**
- File upload/download
- File synchronization
- File sharing
- Version control
- Conflict resolution

**Non-Functional Requirements:**
- 500M users
- Average 1TB per user
- File size up to 50GB
- 99.99% durability

### File Chunking and Deduplication

```python
class DistributedFileStorage:
    def __init__(self):
        self.chunk_size = 4 * 1024 * 1024  # 4MB chunks
        
    def upload_file(self, file_path, user_id):
        """Upload file with chunking and deduplication"""
        
        file_metadata = {
            'file_id': str(uuid.uuid4()),
            'name': os.path.basename(file_path),
            'size': os.path.getsize(file_path),
            'chunks': [],
            'created_at': time.time()
        }
        
        # Chunk file
        chunks = self.chunk_file(file_path)
        
        for chunk_data in chunks:
            # Calculate chunk hash
            chunk_hash = hashlib.sha256(chunk_data).hexdigest()
            
            # Check if chunk already exists (deduplication)
            if not self.chunk_exists(chunk_hash):
                # Upload new chunk
                self.upload_chunk(chunk_hash, chunk_data)
            else:
                # Increment reference count
                self.increment_chunk_reference(chunk_hash)
            
            file_metadata['chunks'].append({
                'hash': chunk_hash,
                'size': len(chunk_data)
            })
        
        # Store file metadata
        self.store_file_metadata(user_id, file_metadata)
        
        return file_metadata['file_id']
    
    def chunk_file(self, file_path):
        """Split file into chunks"""
        
        chunks = []
        
        with open(file_path, 'rb') as f:
            while True:
                chunk = f.read(self.chunk_size)
                if not chunk:
                    break
                chunks.append(chunk)
        
        return chunks
    
    def download_file(self, file_id, user_id):
        """Download and reconstruct file"""
        
        # Get file metadata
        metadata = self.get_file_metadata(user_id, file_id)
        
        # Download chunks
        file_data = b''
        for chunk_info in metadata['chunks']:
            chunk_data = self.download_chunk(chunk_info['hash'])
            file_data += chunk_data
        
        return file_data
    
    def sync_files(self, user_id, client_state):
        """Sync files between client and server"""
        
        # Get server state
        server_state = self.get_user_files(user_id)
        
        # Calculate diff
        to_upload = []
        to_download = []
        conflicts = []
        
        for file_path, client_metadata in client_state.items():
            if file_path not in server_state:
                # New file on client
                to_upload.append(file_path)
            elif client_metadata['modified'] > server_state[file_path]['modified']:
                # Client has newer version
                if server_state[file_path]['modified'] > client_metadata['last_sync']:
                    # Conflict - both modified
                    conflicts.append(file_path)
                else:
                    to_upload.append(file_path)
        
        for file_path, server_metadata in server_state.items():
            if file_path not in client_state:
                # New file on server
                to_download.append(file_path)
            elif server_metadata['modified'] > client_state[file_path]['modified']:
                # Server has newer version
                to_download.append(file_path)
        
        return {
            'to_upload': to_upload,
            'to_download': to_download,
            'conflicts': conflicts
        }
```

## Summary of Design Patterns

### Common Patterns Across Systems

1. **Caching Strategy**
   - Multi-level caching (CDN, Application, Database)
   - Cache-aside pattern for reads
   - Write-through for critical data

2. **Database Design**
   - SQL for transactional data
   - NoSQL for scale and flexibility
   - Specialized databases for specific needs

3. **Scalability Approach**
   - Horizontal scaling with load balancing
   - Database sharding for large datasets
   - Read replicas for read-heavy workloads

4. **Real-time Updates**
   - WebSockets for bidirectional communication
   - Server-sent events for one-way streaming
   - Message queues for async processing

5. **Consistency Models**
   - Strong consistency for financial data
   - Eventual consistency for social features
   - Tunable consistency based on requirements

### Key Takeaways

1. **Start with requirements** - Functional and non-functional
2. **Estimate capacity early** - Helps drive design decisions
3. **Design for failure** - Every component can fail
4. **Use the right tool** - Different databases for different needs
5. **Cache aggressively** - But invalidate carefully
6. **Monitor everything** - You can't fix what you can't measure
7. **Iterate the design** - Start simple, add complexity as needed

## Next Steps for Interview Preparation

1. **Practice drawing diagrams** - Clear communication is key
2. **Learn to estimate** - Back-of-envelope calculations
3. **Understand trade-offs** - No perfect solutions
4. **Study real systems** - Read engineering blogs
5. **Mock interviews** - Practice explaining your design
6. **Stay current** - Technologies evolve rapidly