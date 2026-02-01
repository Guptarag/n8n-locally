# n8n Docker Setup

This repository contains an n8n deployment using Docker with a scalable queue-based architecture. This setup can be used for both **local development** and **production environments**.

## Architecture Overview

This setup implements a distributed n8n architecture with the following components:

- **Main Process**: Disabled for production (`N8N_DISABLE_PRODUCTION_MAIN_PROCESS=true`)
- **Webhook Handler**: Dedicated container for handling incoming webhooks
- **Worker**: Processes workflow executions from the queue (10 concurrent workers)
- **PostgreSQL**: Database for storing workflows and execution data
- **Redis**: Message queue for distributing work between webhook handler and workers

## Components

### 1. Main n8n Container (`DockerFile`)
- **Base Image**: n8nio/n8n:2.0.2
- **Purpose**: Main n8n instance (production main process disabled)
- **Port**: 5678
- **Features**:
  - Queue-based execution mode
  - PostgreSQL database integration
  - Redis queue integration
  - SMTP email configuration
  - Execution data pruning
  - Metrics enabled

### 2. Webhook Handler (`DockerFile-webhook`)
- **Base Image**: n8nio/n8n:2.0.2
- **Purpose**: Handles incoming webhook requests
- **Port**: 5678
- **Command**: `webhook`
- **Features**:
  - Queue-based execution
  - Connects to shared PostgreSQL and Redis

### 3. Worker (`DockerFile-worker`)
- **Base Image**: n8nio/n8n:2.0.2
- **Purpose**: Processes workflow executions from the queue
- **Port**: 5678
- **Command**: `worker --concurrency=10`
- **Features**:
  - 10 concurrent workflow executions
  - Queue health checks enabled
  - Connects to shared PostgreSQL and Redis

### 4. PostgreSQL (`DockerFile-postgres`)
- **Base Image**: postgres:15
- **Database**: n8n
- **Port**: 5432
- **Credentials**: Configured via environment variables

### 5. Redis (`DockerFile-redis`)
- **Base Image**: redis:7
- **Port**: 6379
- **Purpose**: Message queue for job distribution

## Prerequisites

- Have a docker environment to install locally
- Have basic knowledge of docker


## Configuration

### Environment Variables to Update

Before deploying, update the following sensitive values:

#### Database Configuration
```bash
DB_POSTGRESDB_USER=postgres_name          # Change to your username
DB_POSTGRESDB_PASSWORD=postgres_pasword   # Change to secure password
```

#### Encryption
```bash
N8N_ENCRYPTION_KEY=YOUR_ENCRYPTION_KEY_HERE    # Change to secure random key (use: openssl rand -base64 32)
```

#### SMTP Configuration
for email notifications add following environment variables 
```bash
N8N_EMAIL_MODE=smtp
N8N_SMTP_HOST=smtp.example.com
N8N_SMTP_PORT=587
N8N_SMTP_SSL=false
N8N_SMTP_USER=YOUR_SMTP_USER               # Your SMTP username
N8N_SMTP_PASS=YOUR_SMTP_PASSWORD           # Your SMTP password
N8N_SMTP_SENDER=your-email@example.com     # Your sender email
```

#### Webhook URL
```bash
WEBHOOK_URL=https://your-domain.com        # Your public URL
```

### Production Environment - Remote Servers

For production deployments using remote PostgreSQL and Redis servers, update the following host configurations in all Dockerfiles (`DockerFile`, `DockerFile-webhook`, `DockerFile-worker`):

#### Remote PostgreSQL Configuration
```bash
DB_POSTGRESDB_HOST=your-postgres-server.example.com  # Change from 'postgres' to remote host
DB_POSTGRESDB_PORT=5432                              # Update if using non-standard port
DB_POSTGRESDB_USER=your_production_user              # Production database user
DB_POSTGRESDB_PASSWORD=your_secure_password          # Production database password
```

#### Remote Redis Configuration
```bash
QUEUE_BULL_REDIS_HOST=your-redis-server.example.com  # Change from 'redis' to remote host
QUEUE_BULL_REDIS_PORT=6379                           # Update if using non-standard port
```

**Important Notes for Production**:
- The default values (`postgres` and `redis`) are Docker container names for local development
- For production, replace these with your actual remote server hostnames or IP addresses
- Ensure your remote servers are accessible from your n8n containers
- Consider using connection strings with authentication if your Redis server requires it
- Update SSL/TLS settings as needed for secure connections to remote databases

## Key Features

### Execution Settings
- **Mode**: Queue-based for scalability
- **Timeout**: 1500 seconds max per execution
- **Data Retention**: 168 hours (7 days) for executions, 744 hours (31 days) max age
- **Pruning**: Automatic, keeps max 50,000 executions
- **Save Policy**: All executions (errors, success, progress, manual)

### Queue Configuration
- **Prefix**: `n8n-`
- **Lock Duration**: 60 seconds
- **Lock Renew Time**: 30 seconds
- **Stalled Interval**: 60 seconds

### Database Settings
- **Type**: PostgreSQL
- **Pool Size**: 50 connections
- **Connection Timeout**: 30 seconds
- **Idle Timeout**: 40 seconds
- **SSL**: Reject unauthorized enabled

### Node Settings
- **Excluded Nodes**: Gmail and Gmail Trigger nodes
- **Timezone**: Asia/Kolkata

#### Excluding Nodes

The `NODES_EXCLUDE` environment variable allows you to hide specific nodes from the n8n UI and prevent them from being executed. This is useful for security, compliance, or organizational requirements.

**Format**: Must be a **JSON string** (not a plain array) containing a JSON array of node names. The outer quotes and escaped inner quotes are required.

**Example Configuration**:
```dockerfile
ENV NODES_EXCLUDE="[\"n8n-nodes-base.gmail\",\"n8n-nodes-base.gmailTrigger\"]"
```

**Current Excluded Nodes**:
- `n8n-nodes-base.gmail` - Gmail node
- `n8n-nodes-base.gmailTrigger` - Gmail Trigger node

**To exclude additional nodes**:
1. Find the node name (format: `n8n-nodes-base.nodeName`)
2. Add it to the JSON array in all three Dockerfiles:
   - `DockerFile` (main container)
   - `DockerFile-worker` (worker container)
   - `DockerFile-webhook` (webhook container)
3. Rebuild the images and restart containers
4. Clear browser cache (Ctrl+Shift+R or Cmd+Shift+R) to see changes

**Common nodes to exclude for security**:
```dockerfile
# Example: Exclude command execution and file system nodes
ENV NODES_EXCLUDE="[\"n8n-nodes-base.executeCommand\",\"n8n-nodes-base.readWriteFile\",\"n8n-nodes-base.localFileTrigger\"]"
```

**Important**: The `NODES_EXCLUDE` setting must be identical across all n8n containers (main, worker, webhook) to ensure consistent behavior.

## Building Images

Build individual images:

```bash
# Main n8n container
docker build -f DockerFile -t n8n-main:latest .

# Webhook handler
docker build -f DockerFile-webhook -t n8n-webhook:latest .

# Worker
docker build -f DockerFile-worker -t n8n-worker:latest .

# PostgreSQL
docker build -f DockerFile-postgres -t n8n-postgres:latest .

# Redis
docker build -f DockerFile-redis -t n8n-redis:latest .
```

> **⚠️ Important Note**: Whenever you make changes to any Dockerfile, you must rebuild the respective Docker image using the build command above and then restart the container for the changes to take effect.

## Running the Stack

### Manual Docker Commands

1. **Create Docker Network**:
```bash
docker network create n8n-network
```

2. **Start Redis**:
```bash
docker run -d --name n8n-redis --network n8n-network -p 6379:6379 n8n-redis:latest
```

3. **Start PostgreSQL**:
```bash
docker run -d --name n8n-postgres --network n8n-network -p 5432:5432 n8n-postgres:latest
```

4. **Start Webhook Handler**:
```bash
docker run -d --name n8n-webhook --network n8n-network -p 5678:5678 n8n-webhook:latest
```

5. **Start Worker(s)**:
```bash
docker run -d --name n8n-worker-1 --network n8n-network n8n-worker:latest
```

## Scaling

To add more workers for increased throughput:

```bash
docker run -d --name n8n-worker-2 --network n8n-network n8n-worker:latest
docker run -d --name n8n-worker-3 --network n8n-network n8n-worker:latest
# Add as many workers as needed
```

## Monitoring

- **Metrics**: Enabled on all containers (`N8N_METRICS=true`)
- **Log Level**: Info
- **Health Checks**: Queue health checks enabled on workers

## Security Considerations

1. **Change default credentials** in all Dockerfiles before deployment
2. **Use strong encryption keys** for `N8N_ENCRYPTION_KEY`
3. **Secure task runner tokens** (`N8N_RUNNERS_AUTH_TOKEN`)
4. **Enable SSL/TLS** for production deployments
5. **Use secrets management** for sensitive environment variables
6. **Restrict network access** to PostgreSQL and Redis
7. **Keep images updated** regularly for security patches

## Troubleshooting

### Check Container Logs
```bash
docker logs n8n-webhook
docker logs n8n-worker-1
docker logs n8n-postgres
docker logs n8n-redis
```

### Verify Queue Connection
Check Redis connection:
```bash
docker exec -it n8n-redis redis-cli
> KEYS n8n-*
```

### Database Connection
Check PostgreSQL:
```bash
docker exec -it n8n-postgres psql -U postgres_name -d n8n
```

## Performance Tuning

- **Worker Concurrency**: Adjust `--concurrency=10` in `DockerFile-worker`
- **Database Pool Size**: Modify `DB_POSTGRESDB_POOL_SIZE` (default: 50)
- **Memory**: Adjust `NODE_OPTIONS=--max_old_space_size=768` for memory limits
- **Queue Settings**: Tune lock duration and stalled intervals based on workflow complexity

## Support

For n8n documentation and support:
- [n8n Documentation](https://docs.n8n.io/)
- [n8n Community Forum](https://community.n8n.io/)
- [n8n GitHub](https://github.com/n8n-io/n8n)

## License

This configuration is provided as-is. n8n is licensed under the [Sustainable Use License](https://github.com/n8n-io/n8n/blob/master/LICENSE.md).
