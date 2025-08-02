# Stock Market Data & News Aggregator

A Flask-based web application that provides real-time stock market data and related news articles using Alpha Vantage and NewsAPI.

## Features

- Real-time stock price data from Alpha Vantage API
- Financial news aggregation from NewsAPI
- Responsive web interface
- Caching for improved performance
- Load balancer support with Nginx
- Production-ready deployment configurations

## Prerequisites

- Python 3.8 or higher
- pip (Python package installer)
- Web server (Nginx recommended for production)
- API Keys:
  - Alpha Vantage API Key (get from <https://www.alphavantage.co/support/#api-key>)
  - NewsAPI Key (get from <https://newsapi.org/register>)

## Installation & Configuration

### 1. Clone and Setup Environment

```bash
git clone <repository-url>
cd Stock-Market-News

# Create virtual environment
python -m venv venv

# Activate virtual environment
# On Windows:
venv\Scripts\activate
# On Linux/Mac:
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt
```

### 2. Environment Configuration

```bash
# Copy environment template
cp .env.example .env

# Edit .env file with your API keys
# Required variables:
ALPHA_VANTAGE_API_KEY=your_alpha_vantage_api_key_here
NEWS_API_KEY=your_news_api_key_here
PORT=8080
```

## Deployment Options

### Option 1: Simple Development Deployment

**Quick Start (Windows):**

```cmd
start.bat
```

**Quick Start (Linux/Mac):**

```bash
./start.sh
```

**Manual Start:**

```bash
python app.py
```

Application will be available at: <http://localhost:8080>

### Option 2: Production Deployment with Gunicorn

**Install Gunicorn:**

```bash
pip install gunicorn
```

**Start with Gunicorn:**

```bash
gunicorn --bind 0.0.0.0:8080 --workers 4 wsgi:app
```

**Background Process:**

```bash
nohup gunicorn --bind 0.0.0.0:8080 --workers 4 wsgi:app > app.log 2>&1 &
```

### Option 3: Load Balanced Production Deployment

#### Step 1: Deploy Multiple Application Instances

**Server 1 (web01):**

```bash
# Clone repository
git clone <repository-url>
cd Stock-Market-News

# Setup environment
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with your API keys

# Start application on port 8080
gunicorn --bind 0.0.0.0:8080 --workers 4 wsgi:app
```

**Server 2 (web02):**

```bash
# Repeat same steps as Server 1
# Ensure same .env configuration
gunicorn --bind 0.0.0.0:8080 --workers 4 wsgi:app
```

#### Step 2: Configure Nginx Load Balancer

**Install Nginx:**

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nginx

# CentOS/RHEL
sudo yum install nginx
```

**Configure Nginx:**

```bash
# Copy provided configuration
sudo cp nginx.conf /etc/nginx/sites-available/stock-app
sudo ln -s /etc/nginx/sites-available/stock-app /etc/nginx/sites-enabled/

# Edit configuration
sudo nano /etc/nginx/sites-available/stock-app
```

**Update nginx.conf with your server IPs:**

```nginx
upstream stock_app {
    server 192.168.1.10:8080;  # Replace with web01 IP
    server 192.168.1.11:8080;  # Replace with web02 IP
}

server {
    listen 80;
    server_name your-domain.com;  # Replace with your domain
    
    location / {
        proxy_pass http://stock_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
    
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

**Start Nginx:**

```bash
# Test configuration
sudo nginx -t

# Start/restart Nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### Option 4: Systemd Service (Linux)

**Create service file:**

```bash
sudo nano /etc/systemd/system/stock-app.service
```

**Service configuration:**

```ini
[Unit]
Description=Stock Market Data & News Aggregator
After=network.target

[Service]
Type=exec
User=www-data
Group=www-data
WorkingDirectory=/path/to/Stock-Market-News
Environment=PATH=/path/to/Stock-Market-News/venv/bin
ExecStart=/path/to/Stock-Market-News/venv/bin/gunicorn --bind 0.0.0.0:8080 --workers 4 wsgi:app
Restart=always

[Install]
WantedBy=multi-user.target
```

**Enable and start service:**

```bash
sudo systemctl daemon-reload
sudo systemctl enable stock-app
sudo systemctl start stock-app
sudo systemctl status stock-app
```

## Server Configuration Details

### Firewall Configuration

**Ubuntu/Debian (UFW):**

```bash
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 8080/tcp  # Application (if direct access needed)
sudo ufw enable
```

**CentOS/RHEL (firewalld):**

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### SSL/HTTPS Configuration (Optional)

**Install Certbot:**

```bash
sudo apt install certbot python3-certbot-nginx
```

**Obtain SSL certificate:**

```bash
sudo certbot --nginx -d your-domain.com
```

### Performance Tuning

**Gunicorn Configuration:**

```bash
# Create gunicorn.conf.py
workers = 4
worker_class = "sync"
worker_connections = 1000
max_requests = 1000
max_requests_jitter = 100
timeout = 30
keepalive = 2
bind = "0.0.0.0:8080"

# Start with config file
gunicorn -c gunicorn.conf.py wsgi:app
```

## Testing Deployment

### 1. Basic Functionality Test

```bash
# Test application health
curl http://localhost:8080/

# Test stock API endpoint
curl http://localhost:8080/api/stock/AAPL

# Test news API endpoint
curl http://localhost:8080/api/news
```

### 2. Load Balancer Test

```bash
# Test through load balancer
curl http://your-domain.com/

# Test health endpoint
curl http://your-domain.com/health

# Test multiple requests to verify load balancing
for i in {1..10}; do curl -s http://your-domain.com/api/stock/AAPL; done
```

### 3. Automated Testing

**Run test suite:**

```bash
# Install test dependencies
pip install pytest requests

# Run tests
python -m pytest test_app.py -v

# Run deployment tests
python test-deployment.py
```

### 4. Performance Testing

**Using Apache Bench:**

```bash
# Install ab
sudo apt install apache2-utils

# Test concurrent requests
ab -n 100 -c 10 http://localhost:8080/
```

**Using curl for response time:**

```bash
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:8080/api/stock/AAPL
```

## Monitoring & Maintenance

### Log Management

**Application logs:**

```bash
# View Gunicorn logs
tail -f /var/log/gunicorn/access.log
tail -f /var/log/gunicorn/error.log

# View systemd service logs
sudo journalctl -u stock-app -f
```

**Nginx logs:**

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### Health Monitoring

**Create monitoring script:**

```bash
#!/bin/bash
# health-check.sh
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/)
if [ $response -eq 200 ]; then
    echo "Application is healthy"
else
    echo "Application is down - HTTP $response"
    # Add restart logic or alerts here
fi
```

### Backup & Updates

**Application updates:**

```bash
# Pull latest code
git pull origin main

# Update dependencies
pip install -r requirements.txt

# Restart application
sudo systemctl restart stock-app
```

## Troubleshooting

### Common Issues

1. **API Key Errors:**
   - Verify .env file exists and contains valid API keys
   - Check API key quotas and limits

2. **Port Conflicts:**
   - Change PORT in .env file
   - Check for processes using port: `netstat -tulpn | grep :8080`

3. **Permission Issues:**
   - Ensure proper file permissions: `chmod +x start.sh`
   - Check user permissions for service files

4. **Network Issues:**
   - Verify firewall rules
   - Check DNS resolution
   - Test API connectivity: `curl https://www.alphavantage.co/query?function=GLOBAL_QUOTE&symbol=AAPL&apikey=demo`

### Debug Mode

**Enable debug logging:**

```python
# In app.py, temporarily set:
app.run(host='0.0.0.0', port=port, debug=True)
```

**Check application status:**

```bash
# Process status
ps aux | grep gunicorn

# Port usage
sudo netstat -tulpn | grep :8080

# Service status
sudo systemctl status stock-app
```

## API Endpoints

- `GET /` - Main application interface
- `GET /api/stock/<symbol>` - Get stock data for specific symbol
- `GET /api/news` - Get general financial news
- `GET /api/news/<symbol>` - Get news for specific stock symbol
- `POST /get_stock_data` - Get multiple stocks data (JSON payload)
- `GET /health` - Health check endpoint

## Security Considerations

- API keys stored in environment variables
- CORS enabled for cross-origin requests
- Request timeouts configured
- Rate limiting recommended for production
- HTTPS strongly recommended for production
- Regular security updates for dependencies

## Support

For deployment issues:

1. Check application logs
2. Verify API key configuration
3. Test network connectivity
4. Review firewall settings
5. Validate server resources (CPU, memory, disk space)
