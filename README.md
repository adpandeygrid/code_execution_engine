# Code Testing with Self-Hosted Piston API

This project uses a self-hosted Piston API to execute code and run test cases without rate limit restrictions.

## Features

- ✅ Self-hosted Piston API with configurable rate limits
- ✅ NGINX reverse proxy with rate limiting (100 req/s default)
- ✅ Support for multiple programming languages (C++, Python, Java, Rust, etc.)
- ✅ Async execution with configurable concurrency
- ✅ Automatic test case comparison
- ✅ Detailed execution reports

## Prerequisites

- Docker and Docker Compose installed
- Python 3.8 or higher
- Internet connection (for initial Docker image pull)

**Note for Apple Silicon (M1/M2/M3) users**: The docker-compose.yml is configured to use AMD64 platform emulation, which works seamlessly on ARM64 systems.

## Quick Start

### 1. Deploy Piston API with Rate Limiting

The setup includes **NGINX reverse proxy with rate limiting** (100 requests/second by default).

Start the self-hosted Piston API using Docker Compose:

```bash
docker-compose up -d
```

This will:
- Pull the latest Piston API image
- Start NGINX reverse proxy with rate limiting (200 req/s)
- Start the API server (accessible through NGINX on port 2000)
- Create persistent volumes for packages and jobs

**Rate Limiting**: The default configuration allows **200 requests per second** with a burst capacity of 400 requests. See the [Rate Limiting Configuration](#rate-limiting-configuration) section to customize.

### 2. Verify Piston API is Running

Check if the API is healthy:

```bash
curl http://localhost:2000/api/v2/healthz
```

You should see a response indicating the API is running.

### 3. Install Language Runtimes

**Important**: Before running tests, you need to install the language runtimes you'll be using. The Piston API starts with no runtimes installed.

**Manual installation using Piston CLI:**

1. Clone the Piston repository:
   ```bash
   cd /tmp
   git clone https://github.com/engineer-man/piston.git
   cd piston/cli
   npm install
   ```

2. Install runtimes:
   ```bash
   # Set the API URL
   export PISTON_SERVER_URL="http://localhost:2000"
   
   # Install C++ (use "gcc" not "cpp")
   node index.js ppman install gcc
   
   # Install Python
   node index.js ppman install python
   ```

**Note**: Package names are:
- `gcc` for C/C++ (not "cpp")
- `python` for Python
- `java` for Java

**Verify installed runtimes**:

```bash
curl http://localhost:2000/api/v2/runtimes
```

This should return a list of installed runtimes. If it's empty `[]`, runtimes haven't been installed yet.

**Note**: If runtime installation doesn't work, the code will automatically fall back to the public Piston API (with rate limits).

### 4. Install Python Dependencies

```bash
pip install -r requirements.txt
```

### 5. Run Your Tests

```bash
python main.py
```

The script will:
- Read test cases from the `test_cases/` directory
- Execute your code against each test case
- Compare outputs and display results

## Configuration

### Piston API URL

By default, the script connects to `http://localhost:2000`. To use a different URL, set the environment variable:

```bash
export PISTON_API_URL=http://your-piston-server:2000
python main.py
```

### Concurrency Settings

You can adjust concurrency in the `main()` function or when calling `run_test_cases_from_files()`:

```python
results, total_execution_time = await run_test_cases_from_files(
    language, 
    user_source_code, 
    test_cases_dir,
    max_concurrent=10,      # Number of concurrent requests (default: 10)
    requests_per_second=20  # Requests per second limit (default: 20)
)
```

**Or use environment variables:**
```bash
export MAX_CONCURRENT=15
export REQUESTS_PER_SECOND=30
python main.py
```

With a self-hosted instance, you can use much higher values than the public API!

## Project Structure

```
.
├── main.py                 # Main testing script
├── docker-compose.yml      # Piston API deployment configuration
├── requirements.txt        # Python dependencies
├── README.md              # This file
└── test_cases/            # Test case files
    ├── input1.txt
    ├── output1.txt
    ├── input2.txt
    ├── output2.txt
    └── ...
```

## Supported Languages

The following languages are supported (and more can be added to Piston):

- Python (python, python3)
- C++ (cpp, c++)
- C (c)
- Java (java)
- Rust (rust)
- JavaScript (javascript, js)
- Go (go)
- Ruby (ruby)
- PHP (php)
- Swift (swift)
- Kotlin (kotlin)
- Scala (scala)
- R (r)
- Bash (bash, sh)

## Modifying the Code

Edit the `main()` function in `main.py` to change:

1. **Source Code**: Update `user_source_code` with your solution
2. **Language**: Change the `language` variable (e.g., "cpp", "python", "java")
3. **Test Cases Directory**: Modify `test_cases_dir` if needed

## Stopping the Piston API

To stop the Piston API:

```bash
docker-compose down
```

To stop and remove volumes (cleanup):

```bash
docker-compose down -v
```

## Troubleshooting

### Piston API not responding

1. Check if the container is running:
   ```bash
   docker ps
   ```

2. Check container logs:
   ```bash
   docker-compose logs piston
   ```

3. Restart the service:
   ```bash
   docker-compose restart
   ```

### Connection errors

- Ensure the Piston API is running on the expected port (default: 2000)
- Check firewall settings if accessing remotely
- Verify the `PISTON_API_URL` environment variable is correct

### Platform/Architecture errors

If you see platform mismatch errors (e.g., "linux/amd64 does not match linux/arm64"):
- The docker-compose.yml already includes `platform: linux/amd64` for compatibility
- Docker Desktop on Apple Silicon will automatically use emulation
- If issues persist, ensure Docker Desktop is updated to the latest version

### Rate limit errors (unlikely with self-hosted)

If you still see rate limit errors:
- Increase the delay between requests in `run_test_cases_from_files()`
- Reduce `max_concurrent` value
- Check your server resources (CPU, memory)

## Rate Limiting Configuration

The setup includes **two rate limiting approaches**:

### Approach 1: NGINX Reverse Proxy (Currently Active)

The `docker-compose.yml` includes NGINX with rate limiting configured. This is the **recommended approach** as it:
- Doesn't require modifying Piston source code
- Provides better performance and control
- Easy to adjust rate limits

**Current Configuration** (in `nginx.conf`):
- **Rate**: 200 requests per second
- **Burst**: 400 requests (allows temporary spikes)
- **Window**: 1 second

**To Change Rate Limits**:

Edit `nginx.conf` and modify these lines:

```nginx
# Change rate from 200r/s to your desired rate (e.g., 500r/s)
limit_req_zone $binary_remote_addr zone=piston_zone:10m rate=500r/s;

# In the location block, adjust burst (e.g., 1000 for 500r/s)
limit_req zone=piston_zone burst=1000 nodelay;
```

Then restart NGINX:
```bash
docker-compose restart nginx
```

### Approach 2: Express Rate Limiting (Alternative)

If you prefer to add rate limiting directly in the Piston API code:

1. Build a custom Docker image using `Dockerfile.piston`
2. The `apply-rate-limit.js` script will automatically add `express-rate-limit` to the API
3. Default: 100 requests per second

**To use this approach**:
```bash
# Build custom image
docker build -f Dockerfile.piston -t piston-custom .

# Update docker-compose.yml to use piston-custom image
# Then start services
docker-compose up -d
```

**Recommended Rate Limits**:
- **Development**: 100-200 req/s (current default: 200 req/s)
- **Production**: 200-500 req/s  
- **High Throughput**: 500-1000 req/s (adjust based on server capacity)

## Performance Tips

With a self-hosted instance, you can:

- **Increase concurrency**: Use `max_concurrent=10` or higher
- **Reduce delays**: Set `requests_per_second=20` or higher (but respect NGINX rate limits)
- **Scale horizontally**: Deploy multiple Piston instances behind a load balancer
- **Adjust rate limits**: Modify `nginx.conf` to match your server capacity

## Security Considerations

⚠️ **Important**: The Piston API executes arbitrary code. When deploying:

1. **Network Security**: Only expose the API to trusted networks
2. **Firewall Rules**: Restrict access to necessary IPs only
3. **Authentication**: Consider adding authentication if exposing publicly
4. **Resource Limits**: Configure Docker resource limits to prevent abuse

## Advanced Deployment

### Production Deployment

For production use:

1. Use a reverse proxy (nginx/traefik) for SSL/TLS
2. Set up monitoring (Prometheus/Grafana)
3. Configure resource limits in docker-compose.yml
4. Use environment variables for configuration
5. Set up log aggregation

### Example Production docker-compose.yml

```yaml
version: '3.8'

services:
  piston:
    image: ghcr.io/engineer-man/piston:latest
    container_name: piston-api
    ports:
      - "2000:2000"
    volumes:
      - piston_packages:/piston/packages
      - piston_jobs:/piston/jobs
    environment:
      - PISTON_REPO_URL=https://github.com/engineer-man/piston
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
    networks:
      - piston-network

volumes:
  piston_packages:
  piston_jobs:

networks:
  piston-network:
    driver: bridge
```

## License

This project uses the Piston API, which is open-source. Check the Piston repository for license information.

## Support

For issues with:
- **This script**: Check the code comments and error messages
- **Piston API**: Visit [Piston GitHub Repository](https://github.com/engineer-man/piston)
- **Docker**: Refer to Docker documentation

