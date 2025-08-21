# System Design Study Guide - Part 5: Communication and Messaging

## 1. Message Queues

### Understanding Message Queues

Message queues enable asynchronous communication between services by storing messages until the receiving service is ready to process them.

**Core Concepts:**
- **Producer**: Service that sends messages
- **Consumer**: Service that receives and processes messages
- **Queue**: Buffer that stores messages
- **Message**: Data packet with payload and metadata

### Message Queue Patterns

**1. Point-to-Point (Queue):**
```python
class PointToPointQueue:
    """
    One producer, one consumer per message
    Each message is consumed exactly once
    """
    def __init__(self):
        self.queue = []
        self.consumers = []
        self.lock = threading.Lock()
    
    def send_message(self, message):
        with self.lock:
            self.queue.append({
                'id': str(uuid.uuid4()),
                'payload': message,
                'timestamp': time.time(),
                'status': 'pending'
            })
    
    def receive_message(self, consumer_id):
        with self.lock:
            for message in self.queue:
                if message['status'] == 'pending':
                    message['status'] = 'processing'
                    message['consumer_id'] = consumer_id
                    return message
            return None
    
    def acknowledge(self, message_id):
        with self.lock:
            self.queue = [m for m in self.queue if m['id'] != message_id]
    
    def nack(self, message_id):
        """Negative acknowledgment - return message to queue"""
        with self.lock:
            for message in self.queue:
                if message['id'] == message_id:
                    message['status'] = 'pending'
                    del message['consumer_id']
```

**2. Work Queue (Task Distribution):**
```python
class WorkQueue:
    """
    Multiple workers process messages from same queue
    Load balancing across workers
    """
    def __init__(self, num_workers=5):
        self.queue = queue.Queue()
        self.workers = []
        self.results = {}
        
        for i in range(num_workers):
            worker = threading.Thread(target=self._worker, args=(i,))
            worker.daemon = True
            worker.start()
            self.workers.append(worker)
    
    def _worker(self, worker_id):
        while True:
            task = self.queue.get()
            if task is None:
                break
            
            try:
                result = self.process_task(task, worker_id)
                self.results[task['id']] = {
                    'status': 'completed',
                    'result': result,
                    'worker_id': worker_id
                }
            except Exception as e:
                self.results[task['id']] = {
                    'status': 'failed',
                    'error': str(e),
                    'worker_id': worker_id
                }
            finally:
                self.queue.task_done()
    
    def submit_task(self, task):
        task_id = str(uuid.uuid4())
        self.queue.put({
            'id': task_id,
            'payload': task,
            'submitted_at': time.time()
        })
        return task_id
    
    def process_task(self, task, worker_id):
        # Simulate work
        print(f"Worker {worker_id} processing task {task['id']}")
        time.sleep(random.uniform(0.1, 1.0))
        return f"Processed by worker {worker_id}"
```

### RabbitMQ Implementation

**Basic Producer/Consumer:**
```python
import pika
import json

class RabbitMQProducer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
    
    def publish(self, queue_name, message, durable=True):
        # Declare queue (idempotent)
        self.channel.queue_declare(queue=queue_name, durable=durable)
        
        # Publish message
        self.channel.basic_publish(
            exchange='',
            routing_key=queue_name,
            body=json.dumps(message),
            properties=pika.BasicProperties(
                delivery_mode=2  # Make message persistent
            )
        )
    
    def close(self):
        self.connection.close()

class RabbitMQConsumer:
    def __init__(self, host='localhost'):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters(host)
        )
        self.channel = self.connection.channel()
    
    def consume(self, queue_name, callback, auto_ack=False):
        self.channel.queue_declare(queue=queue_name, durable=True)
        
        # Fair dispatch - don't give more than 1 message to worker at a time
        self.channel.basic_qos(prefetch_count=1)
        
        def wrapper(ch, method, properties, body):
            try:
                message = json.loads(body)
                callback(message)
                
                if not auto_ack:
                    ch.basic_ack(delivery_tag=method.delivery_tag)
            except Exception as e:
                print(f"Error processing message: {e}")
                if not auto_ack:
                    # Requeue message
                    ch.basic_nack(
                        delivery_tag=method.delivery_tag,
                        requeue=True
                    )
        
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=wrapper,
            auto_ack=auto_ack
        )
        
        self.channel.start_consuming()
```

**RabbitMQ Exchange Types:**
```python
class RabbitMQExchanges:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
    
    def direct_exchange(self, exchange_name, routing_key, message):
        """
        Routes messages to queues based on exact routing key match
        """
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type='direct'
        )
        
        self.channel.basic_publish(
            exchange=exchange_name,
            routing_key=routing_key,
            body=message
        )
    
    def fanout_exchange(self, exchange_name, message):
        """
        Broadcasts messages to all bound queues
        """
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type='fanout'
        )
        
        self.channel.basic_publish(
            exchange=exchange_name,
            routing_key='',  # Ignored for fanout
            body=message
        )
    
    def topic_exchange(self, exchange_name, routing_key, message):
        """
        Routes based on routing key patterns
        * = exactly one word
        # = zero or more words
        """
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type='topic'
        )
        
        # Examples: 
        # "stock.usd.nyse" matches "stock.*.nyse" or "stock.#"
        self.channel.basic_publish(
            exchange=exchange_name,
            routing_key=routing_key,
            body=message
        )
    
    def headers_exchange(self, exchange_name, headers, message):
        """
        Routes based on message headers
        """
        self.channel.exchange_declare(
            exchange=exchange_name,
            exchange_type='headers'
        )
        
        self.channel.basic_publish(
            exchange=exchange_name,
            routing_key='',
            body=message,
            properties=pika.BasicProperties(headers=headers)
        )
```

### Amazon SQS

**SQS Implementation:**
```python
import boto3
import json

class SQSQueue:
    def __init__(self, queue_name, region='us-east-1'):
        self.sqs = boto3.client('sqs', region_name=region)
        self.queue_name = queue_name
        
        # Create queue if it doesn't exist
        response = self.sqs.create_queue(
            QueueName=queue_name,
            Attributes={
                'DelaySeconds': '0',
                'MessageRetentionPeriod': '86400',  # 1 day
                'VisibilityTimeout': '30',  # 30 seconds
                'MaximumMessageSize': '262144'  # 256 KB
            }
        )
        self.queue_url = response['QueueUrl']
    
    def send_message(self, message, delay_seconds=0):
        response = self.sqs.send_message(
            QueueUrl=self.queue_url,
            MessageBody=json.dumps(message),
            DelaySeconds=delay_seconds,
            MessageAttributes={
                'timestamp': {
                    'StringValue': str(time.time()),
                    'DataType': 'Number'
                }
            }
        )
        return response['MessageId']
    
    def send_batch(self, messages):
        """Send up to 10 messages in one request"""
        entries = []
        for i, msg in enumerate(messages[:10]):
            entries.append({
                'Id': str(i),
                'MessageBody': json.dumps(msg)
            })
        
        response = self.sqs.send_message_batch(
            QueueUrl=self.queue_url,
            Entries=entries
        )
        return response
    
    def receive_messages(self, max_messages=1, wait_time=20):
        """Long polling for messages"""
        response = self.sqs.receive_message(
            QueueUrl=self.queue_url,
            MaxNumberOfMessages=max_messages,
            WaitTimeSeconds=wait_time,  # Long polling
            MessageAttributeNames=['All']
        )
        
        messages = response.get('Messages', [])
        return messages
    
    def delete_message(self, receipt_handle):
        self.sqs.delete_message(
            QueueUrl=self.queue_url,
            ReceiptHandle=receipt_handle
        )
    
    def change_visibility(self, receipt_handle, timeout):
        """Extend processing time for a message"""
        self.sqs.change_message_visibility(
            QueueUrl=self.queue_url,
            ReceiptHandle=receipt_handle,
            VisibilityTimeout=timeout
        )
```

**SQS with Dead Letter Queue:**
```python
class SQSWithDLQ:
    def __init__(self, main_queue, dlq_name):
        self.sqs = boto3.client('sqs')
        
        # Create DLQ
        dlq_response = self.sqs.create_queue(QueueName=dlq_name)
        dlq_url = dlq_response['QueueUrl']
        dlq_arn = self.sqs.get_queue_attributes(
            QueueUrl=dlq_url,
            AttributeNames=['QueueArn']
        )['Attributes']['QueueArn']
        
        # Create main queue with redrive policy
        main_response = self.sqs.create_queue(
            QueueName=main_queue,
            Attributes={
                'RedrivePolicy': json.dumps({
                    'deadLetterTargetArn': dlq_arn,
                    'maxReceiveCount': '3'  # Move to DLQ after 3 failures
                })
            }
        )
        
        self.main_queue_url = main_response['QueueUrl']
        self.dlq_url = dlq_url
```

## 2. Pub/Sub Systems

### Publish-Subscribe Pattern

```python
class PubSubBroker:
    def __init__(self):
        self.topics = {}  # topic -> list of subscribers
        self.lock = threading.RLock()
    
    def create_topic(self, topic_name):
        with self.lock:
            if topic_name not in self.topics:
                self.topics[topic_name] = []
    
    def subscribe(self, topic_name, callback):
        with self.lock:
            if topic_name not in self.topics:
                self.create_topic(topic_name)
            
            subscription_id = str(uuid.uuid4())
            self.topics[topic_name].append({
                'id': subscription_id,
                'callback': callback,
                'subscribed_at': time.time()
            })
            return subscription_id
    
    def unsubscribe(self, topic_name, subscription_id):
        with self.lock:
            if topic_name in self.topics:
                self.topics[topic_name] = [
                    sub for sub in self.topics[topic_name]
                    if sub['id'] != subscription_id
                ]
    
    def publish(self, topic_name, message):
        with self.lock:
            if topic_name not in self.topics:
                return 0
            
            subscribers = self.topics[topic_name].copy()
        
        # Notify subscribers asynchronously
        delivered = 0
        for subscriber in subscribers:
            try:
                # Execute callback in separate thread
                threading.Thread(
                    target=subscriber['callback'],
                    args=(message,)
                ).start()
                delivered += 1
            except Exception as e:
                print(f"Failed to deliver to subscriber {subscriber['id']}: {e}")
        
        return delivered
```

### Redis Pub/Sub

```python
import redis
import threading

class RedisPubSub:
    def __init__(self, host='localhost', port=6379):
        self.redis_client = redis.Redis(host=host, port=port, decode_responses=True)
        self.pubsub = self.redis_client.pubsub()
        self.subscriptions = {}
    
    def publish(self, channel, message):
        """Publish message to channel"""
        return self.redis_client.publish(channel, json.dumps(message))
    
    def subscribe(self, channel, callback):
        """Subscribe to channel with callback"""
        self.pubsub.subscribe(channel)
        
        def listener():
            for message in self.pubsub.listen():
                if message['type'] == 'message':
                    try:
                        data = json.loads(message['data'])
                        callback(data)
                    except Exception as e:
                        print(f"Error processing message: {e}")
        
        # Start listener in background thread
        thread = threading.Thread(target=listener)
        thread.daemon = True
        thread.start()
        
        self.subscriptions[channel] = thread
        return thread
    
    def pattern_subscribe(self, pattern, callback):
        """Subscribe to channels matching pattern"""
        self.pubsub.psubscribe(pattern)
        
        def listener():
            for message in self.pubsub.listen():
                if message['type'] == 'pmessage':
                    try:
                        data = json.loads(message['data'])
                        callback(message['channel'], data)
                    except Exception as e:
                        print(f"Error processing message: {e}")
        
        thread = threading.Thread(target=listener)
        thread.daemon = True
        thread.start()
        return thread

# Usage example
pubsub = RedisPubSub()

# Subscribe to user events
def handle_user_event(data):
    print(f"User event: {data}")

pubsub.subscribe('user:events', handle_user_event)

# Pattern subscription
def handle_order_event(channel, data):
    print(f"Order event on {channel}: {data}")

pubsub.pattern_subscribe('order:*', handle_order_event)

# Publish events
pubsub.publish('user:events', {'type': 'login', 'user_id': '123'})
pubsub.publish('order:created', {'order_id': '456', 'amount': 99.99})
```

## 3. Streaming Platforms

### Apache Kafka

**Kafka Core Concepts:**
- **Topic**: Category for messages
- **Partition**: Ordered, immutable sequence of messages
- **Offset**: Position of message in partition
- **Consumer Group**: Set of consumers sharing work

**Kafka Producer:**
```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json

class KafkaEventProducer:
    def __init__(self, bootstrap_servers=['localhost:9092']):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            acks='all',  # Wait for all replicas
            retries=3,
            max_in_flight_requests_per_connection=1,  # Ensure ordering
            compression_type='gzip'
        )
    
    def send_event(self, topic, event, key=None, partition=None):
        """Send event to Kafka topic"""
        try:
            future = self.producer.send(
                topic,
                value=event,
                key=key,
                partition=partition
            )
            
            # Wait for confirmation (synchronous)
            record_metadata = future.get(timeout=10)
            
            return {
                'topic': record_metadata.topic,
                'partition': record_metadata.partition,
                'offset': record_metadata.offset
            }
        except KafkaError as e:
            print(f"Failed to send event: {e}")
            raise
    
    def send_batch(self, topic, events):
        """Send multiple events efficiently"""
        futures = []
        
        for event in events:
            future = self.producer.send(topic, value=event)
            futures.append(future)
        
        # Flush all events
        self.producer.flush()
        
        # Check results
        results = []
        for future in futures:
            try:
                metadata = future.get(timeout=10)
                results.append({
                    'success': True,
                    'offset': metadata.offset
                })
            except KafkaError as e:
                results.append({
                    'success': False,
                    'error': str(e)
                })
        
        return results
    
    def close(self):
        self.producer.close()
```

**Kafka Consumer:**
```python
from kafka import KafkaConsumer, TopicPartition
from kafka.errors import CommitFailedError

class KafkaEventConsumer:
    def __init__(self, topics, group_id, bootstrap_servers=['localhost:9092']):
        self.consumer = KafkaConsumer(
            *topics,
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            key_deserializer=lambda k: k.decode('utf-8') if k else None,
            enable_auto_commit=False,  # Manual commit for better control
            auto_offset_reset='earliest',  # Start from beginning if no offset
            max_poll_records=100,
            session_timeout_ms=30000
        )
    
    def consume_messages(self, process_callback, batch_size=1):
        """Consume messages with manual commit"""
        batch = []
        
        for message in self.consumer:
            try:
                # Process message
                result = process_callback(message.value)
                
                batch.append(message)
                
                # Commit after batch_size messages
                if len(batch) >= batch_size:
                    self.consumer.commit()
                    batch = []
                    
            except Exception as e:
                print(f"Error processing message: {e}")
                # Optionally handle error (retry, DLQ, etc.)
    
    def consume_with_error_handling(self, process_callback):
        """Consume with error handling and retry logic"""
        max_retries = 3
        
        for message in self.consumer:
            retry_count = 0
            processed = False
            
            while retry_count < max_retries and not processed:
                try:
                    process_callback(message.value)
                    processed = True
                    self.consumer.commit()
                except Exception as e:
                    retry_count += 1
                    if retry_count >= max_retries:
                        # Send to dead letter topic
                        self.send_to_dlq(message)
                    else:
                        time.sleep(2 ** retry_count)  # Exponential backoff
    
    def seek_to_timestamp(self, timestamp):
        """Seek to specific timestamp across all partitions"""
        partitions = self.consumer.partitions_for_topic(self.consumer.topics()[0])
        
        timestamp_dict = {}
        for partition in partitions:
            tp = TopicPartition(self.consumer.topics()[0], partition)
            timestamp_dict[tp] = timestamp
        
        offsets = self.consumer.offsets_for_times(timestamp_dict)
        
        for tp, offset_and_timestamp in offsets.items():
            if offset_and_timestamp:
                self.consumer.seek(tp, offset_and_timestamp.offset)
```

**Kafka Streams Processing:**
```python
class KafkaStreamProcessor:
    def __init__(self, input_topic, output_topic):
        self.input_topic = input_topic
        self.output_topic = output_topic
        self.consumer = KafkaConsumer(
            input_topic,
            bootstrap_servers=['localhost:9092'],
            value_deserializer=lambda m: json.loads(m.decode('utf-8'))
        )
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
    
    def process_stream(self, transform_func):
        """Process stream with transformation"""
        for message in self.consumer:
            try:
                # Transform message
                transformed = transform_func(message.value)
                
                # Send to output topic
                if transformed:
                    self.producer.send(
                        self.output_topic,
                        value=transformed,
                        key=message.key
                    )
                
                # Commit offset
                self.consumer.commit()
                
            except Exception as e:
                print(f"Error processing stream: {e}")
    
    def windowed_aggregation(self, window_seconds=60):
        """Aggregate events in time windows"""
        window = []
        window_start = time.time()
        
        for message in self.consumer:
            window.append(message.value)
            
            # Check if window expired
            if time.time() - window_start >= window_seconds:
                # Process window
                aggregated = self.aggregate_window(window)
                
                # Send aggregated result
                self.producer.send(
                    self.output_topic,
                    value=aggregated
                )
                
                # Reset window
                window = []
                window_start = time.time()
                
                self.consumer.commit()
    
    def aggregate_window(self, events):
        """Aggregate events in window"""
        return {
            'window_start': time.time() - 60,
            'window_end': time.time(),
            'count': len(events),
            'events': events
        }
```

### Amazon Kinesis

```python
import boto3

class KinesisStream:
    def __init__(self, stream_name, region='us-east-1'):
        self.kinesis = boto3.client('kinesis', region_name=region)
        self.stream_name = stream_name
        
        # Create stream if it doesn't exist
        try:
            self.kinesis.create_stream(
                StreamName=stream_name,
                ShardCount=1
            )
            
            # Wait for stream to be active
            waiter = self.kinesis.get_waiter('stream_exists')
            waiter.wait(StreamName=stream_name)
        except self.kinesis.exceptions.ResourceInUseException:
            pass  # Stream already exists
    
    def put_record(self, data, partition_key):
        """Put single record to stream"""
        response = self.kinesis.put_record(
            StreamName=self.stream_name,
            Data=json.dumps(data),
            PartitionKey=partition_key
        )
        
        return {
            'shard_id': response['ShardId'],
            'sequence_number': response['SequenceNumber']
        }
    
    def put_records_batch(self, records):
        """Put multiple records in one request"""
        kinesis_records = []
        for record in records:
            kinesis_records.append({
                'Data': json.dumps(record['data']),
                'PartitionKey': record.get('partition_key', str(uuid.uuid4()))
            })
        
        response = self.kinesis.put_records(
            StreamName=self.stream_name,
            Records=kinesis_records
        )
        
        return {
            'successful': response['Records'],
            'failed_count': response['FailedRecordCount']
        }
    
    def consume_stream(self, process_callback):
        """Consume records from stream"""
        # Get shard iterator
        response = self.kinesis.describe_stream(StreamName=self.stream_name)
        shard_id = response['StreamDescription']['Shards'][0]['ShardId']
        
        shard_iterator_response = self.kinesis.get_shard_iterator(
            StreamName=self.stream_name,
            ShardId=shard_id,
            ShardIteratorType='TRIM_HORIZON'  # Start from beginning
        )
        
        shard_iterator = shard_iterator_response['ShardIterator']
        
        # Consume records
        while shard_iterator:
            response = self.kinesis.get_records(
                ShardIterator=shard_iterator,
                Limit=100
            )
            
            records = response['Records']
            
            for record in records:
                data = json.loads(record['Data'])
                process_callback(data)
            
            shard_iterator = response.get('NextShardIterator')
            
            if not records:
                time.sleep(1)  # No records, wait before next poll
```

## 4. WebSockets and Server-Sent Events

### WebSocket Implementation

```python
import asyncio
import websockets
import json

class WebSocketServer:
    def __init__(self):
        self.clients = set()
        self.rooms = {}  # room_name -> set of clients
    
    async def register_client(self, websocket):
        self.clients.add(websocket)
        await self.send_to_client(websocket, {
            'type': 'connected',
            'message': 'Welcome to WebSocket server'
        })
    
    async def unregister_client(self, websocket):
        self.clients.discard(websocket)
        
        # Remove from all rooms
        for room in self.rooms.values():
            room.discard(websocket)
    
    async def send_to_client(self, websocket, message):
        try:
            await websocket.send(json.dumps(message))
        except websockets.exceptions.ConnectionClosed:
            await self.unregister_client(websocket)
    
    async def broadcast(self, message, exclude=None):
        """Send message to all connected clients"""
        if self.clients:
            tasks = []
            for client in self.clients:
                if client != exclude:
                    tasks.append(self.send_to_client(client, message))
            
            await asyncio.gather(*tasks, return_exceptions=True)
    
    async def join_room(self, websocket, room_name):
        if room_name not in self.rooms:
            self.rooms[room_name] = set()
        
        self.rooms[room_name].add(websocket)
        
        await self.room_broadcast(room_name, {
            'type': 'user_joined',
            'room': room_name
        }, exclude=websocket)
    
    async def room_broadcast(self, room_name, message, exclude=None):
        """Send message to all clients in a room"""
        if room_name in self.rooms:
            tasks = []
            for client in self.rooms[room_name]:
                if client != exclude:
                    tasks.append(self.send_to_client(client, message))
            
            await asyncio.gather(*tasks, return_exceptions=True)
    
    async def handle_client(self, websocket, path):
        await self.register_client(websocket)
        
        try:
            async for message in websocket:
                data = json.loads(message)
                
                if data['type'] == 'join_room':
                    await self.join_room(websocket, data['room'])
                
                elif data['type'] == 'room_message':
                    await self.room_broadcast(
                        data['room'],
                        {
                            'type': 'message',
                            'content': data['content'],
                            'timestamp': time.time()
                        },
                        exclude=websocket
                    )
                
                elif data['type'] == 'broadcast':
                    await self.broadcast(
                        {
                            'type': 'broadcast',
                            'content': data['content']
                        },
                        exclude=websocket
                    )
        
        except websockets.exceptions.ConnectionClosed:
            pass
        finally:
            await self.unregister_client(websocket)
    
    def start_server(self, host='localhost', port=8765):
        start_server = websockets.serve(
            self.handle_client,
            host,
            port
        )
        
        asyncio.get_event_loop().run_until_complete(start_server)
        asyncio.get_event_loop().run_forever()
```

**WebSocket Client:**
```python
class WebSocketClient:
    def __init__(self, uri):
        self.uri = uri
        self.websocket = None
    
    async def connect(self):
        self.websocket = await websockets.connect(self.uri)
    
    async def send_message(self, message):
        await self.websocket.send(json.dumps(message))
    
    async def receive_messages(self, callback):
        async for message in self.websocket:
            data = json.loads(message)
            callback(data)
    
    async def close(self):
        await self.websocket.close()

# Usage
async def main():
    client = WebSocketClient('ws://localhost:8765')
    await client.connect()
    
    # Send message
    await client.send_message({
        'type': 'join_room',
        'room': 'general'
    })
    
    # Receive messages
    def handle_message(data):
        print(f"Received: {data}")
    
    await client.receive_messages(handle_message)
```

### Server-Sent Events (SSE)

```python
from flask import Flask, Response, request
import time

class SSEServer:
    def __init__(self):
        self.app = Flask(__name__)
        self.clients = []
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.route('/events')
        def events():
            def generate():
                client_id = str(uuid.uuid4())
                self.clients.append(client_id)
                
                try:
                    while True:
                        # Send heartbeat every 30 seconds
                        yield f"event: heartbeat\ndata: {time.time()}\n\n"
                        time.sleep(30)
                except GeneratorExit:
                    self.clients.remove(client_id)
            
            return Response(
                generate(),
                mimetype='text/event-stream',
                headers={
                    'Cache-Control': 'no-cache',
                    'X-Accel-Buffering': 'no'  # Disable Nginx buffering
                }
            )
        
        @self.app.route('/broadcast', methods=['POST'])
        def broadcast():
            message = request.json
            self.send_to_all_clients(message)
            return {'status': 'sent'}
    
    def send_to_all_clients(self, data):
        """Send event to all connected clients"""
        message = f"event: message\ndata: {json.dumps(data)}\n\n"
        
        for client in self.clients:
            # In real implementation, you'd need to track client connections
            pass
    
    def format_sse(self, data, event=None, id=None, retry=None):
        """Format message for SSE"""
        message = ''
        
        if event:
            message += f'event: {event}\n'
        
        if id:
            message += f'id: {id}\n'
        
        if retry:
            message += f'retry: {retry}\n'
        
        message += f'data: {json.dumps(data)}\n\n'
        
        return message
```

**SSE Client (JavaScript):**
```javascript
class SSEClient {
    constructor(url) {
        this.eventSource = new EventSource(url);
        this.setupEventHandlers();
    }
    
    setupEventHandlers() {
        // Handle connection open
        this.eventSource.onopen = (event) => {
            console.log('SSE connection opened');
        };
        
        // Handle generic messages
        this.eventSource.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
        
        // Handle named events
        this.eventSource.addEventListener('heartbeat', (event) => {
            console.log('Heartbeat received:', event.data);
        });
        
        this.eventSource.addEventListener('notification', (event) => {
            const notification = JSON.parse(event.data);
            this.showNotification(notification);
        });
        
        // Handle errors
        this.eventSource.onerror = (event) => {
            if (this.eventSource.readyState === EventSource.CLOSED) {
                console.log('SSE connection closed');
            } else {
                console.error('SSE error:', event);
            }
        };
    }
    
    handleMessage(data) {
        console.log('Received message:', data);
    }
    
    close() {
        this.eventSource.close();
    }
}
```

## 5. Long Polling vs Short Polling

### Short Polling

```python
class ShortPollingClient:
    def __init__(self, server_url, poll_interval=5):
        self.server_url = server_url
        self.poll_interval = poll_interval
        self.running = False
    
    def start_polling(self, callback):
        self.running = True
        
        while self.running:
            try:
                # Make request to server
                response = requests.get(f"{self.server_url}/messages")
                
                if response.status_code == 200:
                    messages = response.json()
                    
                    for message in messages:
                        callback(message)
                
                # Wait before next poll
                time.sleep(self.poll_interval)
                
            except Exception as e:
                print(f"Polling error: {e}")
                time.sleep(self.poll_interval)
    
    def stop_polling(self):
        self.running = False

class ShortPollingServer:
    def __init__(self):
        self.message_queues = {}  # client_id -> message queue
    
    def get_messages(self, client_id):
        """Get pending messages for client"""
        if client_id not in self.message_queues:
            return []
        
        messages = []
        queue = self.message_queues[client_id]
        
        # Get all pending messages
        while not queue.empty():
            messages.append(queue.get())
        
        return messages
```

### Long Polling

```python
class LongPollingServer:
    def __init__(self):
        self.message_queues = {}
        self.app = Flask(__name__)
        self.setup_routes()
    
    def setup_routes(self):
        @self.app.route('/poll')
        def long_poll():
            client_id = request.args.get('client_id')
            timeout = int(request.args.get('timeout', 30))
            
            # Wait for messages or timeout
            messages = self.wait_for_messages(client_id, timeout)
            
            return jsonify(messages)
    
    def wait_for_messages(self, client_id, timeout):
        """Wait for messages with timeout"""
        if client_id not in self.message_queues:
            self.message_queues[client_id] = queue.Queue()
        
        client_queue = self.message_queues[client_id]
        messages = []
        end_time = time.time() + timeout
        
        while time.time() < end_time:
            try:
                # Wait for message with remaining timeout
                remaining = end_time - time.time()
                if remaining <= 0:
                    break
                
                message = client_queue.get(timeout=min(remaining, 1))
                messages.append(message)
                
                # Check if more messages available
                while not client_queue.empty():
                    messages.append(client_queue.get_nowait())
                
                # Return immediately if we have messages
                if messages:
                    break
                    
            except queue.Empty:
                continue
        
        return messages
    
    def send_message(self, client_id, message):
        """Send message to client"""
        if client_id not in self.message_queues:
            self.message_queues[client_id] = queue.Queue()
        
        self.message_queues[client_id].put(message)

class LongPollingClient:
    def __init__(self, server_url, client_id):
        self.server_url = server_url
        self.client_id = client_id
        self.running = False
    
    async def start_polling(self, callback):
        self.running = True
        
        while self.running:
            try:
                # Long poll with 30 second timeout
                response = await self.long_poll_request()
                
                if response:
                    for message in response:
                        callback(message)
                
            except Exception as e:
                print(f"Long polling error: {e}")
                await asyncio.sleep(1)  # Brief delay before retry
    
    async def long_poll_request(self):
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.server_url}/poll",
                params={
                    'client_id': self.client_id,
                    'timeout': 30
                }
            ) as response:
                if response.status == 200:
                    return await response.json()
                return None
```

## 6. Protocol Buffers and Data Serialization

### Protocol Buffers

**Proto Definition:**
```protobuf
syntax = "proto3";

package messaging;

message User {
    int64 id = 1;
    string username = 2;
    string email = 3;
    repeated string roles = 4;
    map<string, string> metadata = 5;
}

message Event {
    string event_id = 1;
    string event_type = 2;
    int64 timestamp = 3;
    
    oneof payload {
        UserCreated user_created = 4;
        OrderPlaced order_placed = 5;
        PaymentProcessed payment_processed = 6;
    }
}

message UserCreated {
    User user = 1;
}

message OrderPlaced {
    string order_id = 1;
    int64 user_id = 2;
    repeated OrderItem items = 3;
    double total = 4;
}

message OrderItem {
    string product_id = 1;
    int32 quantity = 2;
    double price = 3;
}
```

**Python Implementation:**
```python
import messaging_pb2  # Generated from protobuf

class ProtobufSerializer:
    def serialize_event(self, event_type, payload):
        event = messaging_pb2.Event()
        event.event_id = str(uuid.uuid4())
        event.event_type = event_type
        event.timestamp = int(time.time() * 1000)
        
        if event_type == 'user_created':
            user_created = messaging_pb2.UserCreated()
            user_created.user.id = payload['user_id']
            user_created.user.username = payload['username']
            user_created.user.email = payload['email']
            user_created.user.roles.extend(payload.get('roles', []))
            
            event.user_created.CopyFrom(user_created)
        
        elif event_type == 'order_placed':
            order_placed = messaging_pb2.OrderPlaced()
            order_placed.order_id = payload['order_id']
            order_placed.user_id = payload['user_id']
            order_placed.total = payload['total']
            
            for item in payload['items']:
                order_item = order_placed.items.add()
                order_item.product_id = item['product_id']
                order_item.quantity = item['quantity']
                order_item.price = item['price']
            
            event.order_placed.CopyFrom(order_placed)
        
        return event.SerializeToString()
    
    def deserialize_event(self, data):
        event = messaging_pb2.Event()
        event.ParseFromString(data)
        
        result = {
            'event_id': event.event_id,
            'event_type': event.event_type,
            'timestamp': event.timestamp
        }
        
        # Extract payload based on type
        which_payload = event.WhichOneof('payload')
        if which_payload:
            payload_obj = getattr(event, which_payload)
            result['payload'] = self.message_to_dict(payload_obj)
        
        return result
```

### Serialization Format Comparison

```python
import json
import pickle
import msgpack
import time

class SerializationBenchmark:
    def __init__(self):
        self.test_data = {
            'user_id': 12345,
            'username': 'john_doe',
            'email': 'john@example.com',
            'metadata': {
                'last_login': time.time(),
                'preferences': {
                    'theme': 'dark',
                    'language': 'en'
                }
            },
            'tags': ['premium', 'verified', 'active']
        }
    
    def benchmark_json(self):
        # Serialize
        start = time.time()
        serialized = json.dumps(self.test_data)
        serialize_time = time.time() - start
        
        # Deserialize
        start = time.time()
        deserialized = json.loads(serialized)
        deserialize_time = time.time() - start
        
        return {
            'format': 'JSON',
            'size': len(serialized),
            'serialize_time': serialize_time,
            'deserialize_time': deserialize_time
        }
    
    def benchmark_pickle(self):
        # Serialize
        start = time.time()
        serialized = pickle.dumps(self.test_data)
        serialize_time = time.time() - start
        
        # Deserialize
        start = time.time()
        deserialized = pickle.loads(serialized)
        deserialize_time = time.time() - start
        
        return {
            'format': 'Pickle',
            'size': len(serialized),
            'serialize_time': serialize_time,
            'deserialize_time': deserialize_time
        }
    
    def benchmark_msgpack(self):
        # Serialize
        start = time.time()
        serialized = msgpack.packb(self.test_data)
        serialize_time = time.time() - start
        
        # Deserialize
        start = time.time()
        deserialized = msgpack.unpackb(serialized)
        deserialize_time = time.time() - start
        
        return {
            'format': 'MessagePack',
            'size': len(serialized),
            'serialize_time': serialize_time,
            'deserialize_time': deserialize_time
        }
```

## 7. Service Mesh and Inter-Service Communication

### Service Mesh Architecture

```python
class ServiceMeshProxy:
    """
    Sidecar proxy for service mesh
    """
    def __init__(self, service_name):
        self.service_name = service_name
        self.circuit_breakers = {}
        self.retry_policies = {}
        self.load_balancers = {}
        self.metrics = {}
    
    def intercept_request(self, target_service, request):
        """Intercept outgoing requests"""
        
        # Apply circuit breaker
        if not self.is_circuit_open(target_service):
            return self.fail_fast(target_service)
        
        # Load balancing
        endpoint = self.get_endpoint(target_service)
        
        # Add tracing headers
        request = self.add_tracing_headers(request)
        
        # Apply retry policy
        response = self.execute_with_retry(
            endpoint,
            request,
            self.retry_policies.get(target_service, {})
        )
        
        # Record metrics
        self.record_metrics(target_service, response)
        
        return response
    
    def execute_with_retry(self, endpoint, request, retry_policy):
        max_retries = retry_policy.get('max_retries', 3)
        backoff = retry_policy.get('backoff', 'exponential')
        
        for attempt in range(max_retries):
            try:
                response = self.make_request(endpoint, request)
                
                if response.status_code < 500:
                    return response
                
            except Exception as e:
                if attempt == max_retries - 1:
                    raise
            
            # Calculate backoff
            if backoff == 'exponential':
                delay = 2 ** attempt
            else:
                delay = 1
            
            time.sleep(delay)
        
        raise Exception(f"Max retries exceeded for {endpoint}")
    
    def add_tracing_headers(self, request):
        """Add distributed tracing headers"""
        if 'X-Request-ID' not in request.headers:
            request.headers['X-Request-ID'] = str(uuid.uuid4())
        
        request.headers['X-B3-TraceId'] = self.get_or_create_trace_id()
        request.headers['X-B3-SpanId'] = str(uuid.uuid4())
        request.headers['X-B3-ParentSpanId'] = self.get_current_span_id()
        
        return request
```

### Istio Configuration Example

```yaml
# VirtualService for traffic management
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-route
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        user-type:
          exact: premium
    route:
    - destination:
        host: reviews
        subset: v2
      weight: 100
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 90
    - destination:
        host: reviews
        subset: v2
      weight: 10

---
# DestinationRule for load balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews-destination
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 100
        h2MaxRequests: 100
    loadBalancer:
      consistentHash:
        httpCookie:
          name: "session-cookie"
          ttl: 3600s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      connectionPool:
        tcp:
          maxConnections: 10
```

## 8. Message Ordering and Delivery Guarantees

### Message Ordering Strategies

```python
class OrderedMessageProcessor:
    def __init__(self):
        self.sequence_numbers = {}  # partition -> last_sequence
        self.pending_messages = {}  # partition -> list of out-of-order messages
    
    def process_message(self, message, partition_key):
        """Process messages in order per partition"""
        
        if partition_key not in self.sequence_numbers:
            self.sequence_numbers[partition_key] = 0
            self.pending_messages[partition_key] = []
        
        expected_sequence = self.sequence_numbers[partition_key] + 1
        message_sequence = message['sequence_number']
        
        if message_sequence == expected_sequence:
            # Process this message
            self.handle_message(message)
            self.sequence_numbers[partition_key] = message_sequence
            
            # Process any pending messages that are now in order
            self.process_pending_messages(partition_key)
            
        elif message_sequence > expected_sequence:
            # Out of order - store for later
            self.pending_messages[partition_key].append(message)
            self.pending_messages[partition_key].sort(
                key=lambda m: m['sequence_number']
            )
        else:
            # Duplicate or old message - ignore
            print(f"Ignoring old message with sequence {message_sequence}")
    
    def process_pending_messages(self, partition_key):
        """Process pending messages that are now in order"""
        while self.pending_messages[partition_key]:
            next_message = self.pending_messages[partition_key][0]
            expected = self.sequence_numbers[partition_key] + 1
            
            if next_message['sequence_number'] == expected:
                self.pending_messages[partition_key].pop(0)
                self.handle_message(next_message)
                self.sequence_numbers[partition_key] = expected
            else:
                break
```

### Delivery Guarantees

```python
class DeliveryGuarantees:
    """
    Implement different delivery guarantee patterns
    """
    
    def at_most_once_delivery(self, message, consumer):
        """
        Fire and forget - may lose messages
        """
        try:
            consumer.process(message)
        except Exception:
            pass  # Don't retry, message is lost
    
    def at_least_once_delivery(self, message, consumer, max_retries=3):
        """
        Retry until success - may duplicate messages
        """
        for attempt in range(max_retries):
            try:
                consumer.process(message)
                return True
            except Exception as e:
                if attempt == max_retries - 1:
                    # Send to DLQ or log error
                    self.send_to_dlq(message, str(e))
                    return False
                time.sleep(2 ** attempt)
        
        return False
    
    def exactly_once_delivery(self, message, consumer):
        """
        Process exactly once using idempotency key
        """
        idempotency_key = message.get('idempotency_key')
        
        if not idempotency_key:
            idempotency_key = hashlib.sha256(
                json.dumps(message, sort_keys=True).encode()
            ).hexdigest()
        
        # Check if already processed
        if self.is_already_processed(idempotency_key):
            return self.get_cached_result(idempotency_key)
        
        # Process with transaction
        with self.database.transaction():
            # Mark as processing
            self.mark_processing(idempotency_key)
            
            try:
                result = consumer.process(message)
                
                # Store result
                self.store_result(idempotency_key, result)
                
                return result
            except Exception as e:
                # Rollback
                self.mark_failed(idempotency_key, str(e))
                raise
```

## Summary and Key Takeaways

1. **Message queues enable decoupling** - Services can operate independently with async communication
2. **Choose between point-to-point and pub/sub** - Based on your communication pattern
3. **Kafka excels at high-throughput streaming** - Use for event streaming and log aggregation
4. **WebSockets for real-time bidirectional** - Chat, gaming, collaborative editing
5. **SSE for server-to-client streaming** - Simpler than WebSockets for one-way communication
6. **Long polling reduces latency vs short polling** - But holds connections longer
7. **Protocol Buffers for efficient serialization** - Smaller size, faster parsing than JSON
8. **Service mesh handles cross-cutting concerns** - Retry, circuit breaking, observability
9. **Message ordering requires careful design** - Use partition keys and sequence numbers
10. **Delivery guarantees have trade-offs** - At-least-once is usually the pragmatic choice

## Next Steps
- Implement a pub/sub system with Redis
- Build a WebSocket chat application
- Set up Kafka for event streaming
- Study Part 6: Scalability and Performance for optimization techniques