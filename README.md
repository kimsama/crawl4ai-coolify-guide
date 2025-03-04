# Deploying Crawl4AI with Coolify: A Complete Guide

This guide walks through the process of deploying the Crawl4AI web crawler service using Coolify, including troubleshooting common issues.

## Prerequisites

- A server with Coolify installed
- Basic knowledge of Docker and web services
- A domain name (optional but recommended)

## Step 1: Initial Setup in Coolify

1. In your Coolify dashboard, create a new service
2. Select "Docker Image" as the deployment type
3. Configure the basic settings:
   - **Name**: `docker-image-crawl4ai` (or your preferred name)
   - **Domains**: Your domain (e.g., `crawl4ai.yourdomain.com`)
   - **Direction**: Allow www & non-www

## Step 2: Configure Docker Settings

1. Set up the Docker image information:
   - **Docker Image**: `unclecode/crawl4ai`
   - **Docker Image Tag**: `basic-amd64`

2. Ensure the image reference format is correct (common error point)
   - ❌ Don't use: `docker pull unclecode/crawl4ai:basic-amd64`
   - ✅ Use: `unclecode/crawl4ai:basic-amd64`

## Step 3: Configure Network Settings

1. Set up port mapping to expose Crawl4AI's service port:
   - **Ports Exposes**: `80,11235`
   - **Port Mappings**: `3000:3000,11235:11235`
   
   > ⚠️ Important: Ensure there are no spaces in the port mappings

2. If needed, add custom Docker options:
   ```
   --cap-add SYS_ADMIN --device=/dev/fuse --security-opt apparmor:unconfined --ulimit nofile=1024:1024 --tmpfs /run/rw:noexec,nosuid,size=65536k
   ```

## Step 4: Set Environment Variables

Add the following environment variable:
- **Name**: `CRAWL4AI_API_TOKEN`
- **Value**: `YOUR_RANDOMLY_CREATED_BEARER_TOKEN`

> ⚠️ Important: Create a strong, random token for security purposes.

## Step 5: Configure Traefik for API Routing

This is the most critical step and often the main source of issues when deploying Crawl4AI with Coolify.

### Understanding the Challenge

The core issue is that Crawl4AI runs its API on port 11235, while Coolify's default Traefik configuration only routes traffic through the standard HTTP/HTTPS ports (80/443) to your application's port 80. We need special Traefik rules to:

1. Recognize API-specific paths (like `/crawl`, `/task`, `/results`)
2. Route these paths to the container's port 11235 instead of the default port 80
3. Ensure these rules take precedence over the default routing rules

### Adding the Traefik Labels

Add these Traefik labels to your container configuration in the "Container Labels" section:

```
traefik.http.routers.crawl4ai-api.rule=Host(`crawl4ai.yourdomain.com`) && (PathPrefix(`/crawl`) || PathPrefix(`/task`) || PathPrefix(`/results`))
traefik.http.routers.crawl4ai-api.priority=100
traefik.http.routers.crawl4ai-api.entryPoints=https
traefik.http.routers.crawl4ai-api.tls=true
traefik.http.routers.crawl4ai-api.tls.certresolver=letsencrypt
traefik.http.routers.crawl4ai-api.service=crawl4ai-api-service
traefik.http.services.crawl4ai-api-service.loadbalancer.server.port=11235
```

### Explanation of Key Components

- **Rule with PathPrefix**: This defines which URLs should be routed to port 11235. We're capturing three API paths:
  - `/crawl` - For submitting crawl jobs
  - `/task` - For checking task status
  - `/results` - For retrieving crawl results
  
- **Priority: 100**: This ensures our API routes take precedence over other routes that might match the same paths

- **loadbalancer.server.port=11235**: This is the critical part that routes matching requests to port 11235 inside the container

### Common Pitfalls

1. **Conflicting Rules**: If you see "Bad Gateway" errors, you may have conflicting Traefik rules. Check if any other rules might be capturing the same paths.

2. **Missing Paths**: If you only define routing for `/crawl` but not for `/task` or `/results`, you'll be able to submit jobs but not check their status.

3. **Order of Labels**: While the order doesn't technically matter, add these labels at the end of your existing Traefik config to avoid overwriting any settings.

4. **Case Sensitivity**: Traefik rules are case-sensitive; ensure paths match exactly what the Crawl4AI API expects.

This configuration routes API-specific paths to port 11235 while keeping the web interface (if any) on the standard ports.

## Step 6: Firewall Configuration

Ensure port 11235 is accessible:

1. If using a cloud provider (AWS, GCP, Azure, etc.):
   - Open port 11235 in your security group/firewall rules
   
2. If using a local firewall (UFW, firewalld, etc.):
   ```bash
   sudo ufw allow 11235/tcp
   ```

## Step 7: Deploy and Verify

1. Deploy your container by clicking "Save" and then "Deploy"
2. Check the deployment logs for any errors
3. Verify the service is running with a health check:

   ```bash
   curl -X GET "https://crawl4ai.yourdomain.com/health"
   ```

   Expected response:
   ```json
   {"status":"healthy","available_slots":5,"memory_usage":38.4,"cpu_usage":7.1}
   ```

## Step 8: Using the Crawl4AI API

Submit a crawl job:

```bash
curl -X POST "https://crawl4ai.yourdomain.com/crawl" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -d '{"urls": ["https://example.com"]}'
```

Check task status:

```bash
curl -X GET "https://crawl4ai.yourdomain.com/task/TASK_ID" \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

Retrieve results:

```bash
curl -X GET "https://crawl4ai.yourdomain.com/results/TASK_ID" \
  -H "Authorization: Bearer YOUR_API_TOKEN"
```

## Common Issues and Troubleshooting

### Invalid Reference Format

**Error**:
```
unable to get image 'docker pull unclecode/crawl4ai:basic-amd64': Error response from daemon: invalid reference format
```

**Solution**: Remove "docker pull" prefix from image name.

### Port Already Allocated

**Error**:
```
Error response from daemon: driver failed programming external connectivity on endpoint: Bind for 0.0.0.0:3000 failed: port is already allocated
```

**Solution**: Change host port to an unused one (e.g., `3001:3000`).

### Not Authenticated

**Error**:
```
{"detail":"Not authenticated"}
```

**Solution**: Include proper authorization header with your API token.

### Bad Gateway Errors

**Error**: 502 Bad Gateway responses when accessing API endpoints.

**Solution**: Ensure Traefik rules properly route ALL API paths to port 11235.

### Field Required Error

**Error**:
```
{"detail":[{"type":"missing","loc":["body","urls"],"msg":"Field required","input":{"url":"https://example.com"}}]}
```

**Solution**: Use proper request format with `urls` as an array.

## Advanced Configuration

For production environments, consider:

1. Setting up health checks
2. Configuring resource limits (CPU, memory)
3. Setting up automatic restarts
4. Configuring volume mounts for persistent data

---

With this guide, you should have a functional Crawl4AI service deployed through Coolify, accessible both internally and externally.