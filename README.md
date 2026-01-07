# MuMo - Multi-Modal Monitoring System

MuMo is a comprehensive monitoring and data visualization platform built with PHP, JavaScript, and RDF technologies. It provides real-time sensor data collection, user management, and interactive dashboards for monitoring various environmental and operational metrics.

## Architecture Overview

MuMo consists of several interconnected components:

- **Dashboard** (PHP/Apache): Web interface for data visualization and system management
- **Pipeline** (Node.js/RDF): Data processing pipeline using RDF Connect framework
- **Database** (MySQL): Data storage for users, sensors, and measurements
- **Nginx**: Reverse proxy and SSL termination

All containers are connected through a shared public Docker network. Nginx is the main entry point and proxies requests to the appropriate backend services.

```
Browser → Nginx → (PHP | Adminer | Solid | Pipeline)
                    ↓
                  MySQL
```

## Project Structure

```
mumo-full/
├── dashboard/           # PHP web application
│   ├── pages/          # PHP pages for different functionalities
│   ├── assets/         # CSS, JS, images
│   ├── handlers/        # API endpoints
│   └── database.sql    # Database schema
├── pipeline/           # RDF data processing pipeline
│   ├── pipeline/       # TTL pipeline definitions
│   ├── proc/          # Custom processors
│   └── Dockerfile     # Node.js container build
├── data/              # LDES data storage
├── nginx.conf         # Nginx configuration
├── docker-compose.yml # Container orchestration
└── settings.php       # Configuration file
```

## Services

### nginx
**Role:** Reverse proxy and public entrypoint.
- **Image:** nginx:alpine
- **Ports:** 80:80, 443:443 (SSL)
- **Configuration:** ./nginx.conf → /etc/nginx/conf.d/default.conf
- **Responsibilities:** Routes requests to backend services, SSL termination

### solid
**Role:** Runs the main RDF/Linked Data pipeline (authentication-oriented).
- **Built with:** RDF-Connect
- **Pipeline:** ./pipeline/pipeline/auth-pipeline.ttl
- **Components:** Community Solid Server, ACL generator, Solid Server Identity Provider
- **Internal port:** 3000
- **Environment variables:**
  - `groupHistory`: http://php/history.php?groups
  - `userHistory`: http://php/history.php?users
  - `sensorHistory`: http://php/history.php?sensors
  - `baseUrl`: Base URL for the application

### pipeline
**Role:** Runs the main data-processing RDF pipeline.
- **Built with:** RDF-Connect
- **Pipeline:** ./pipeline/pipeline/pipeline.ttl
- **Environment variables:**
  - `groupHistory`: http://php/history.php?groups
  - `userHistory`: http://php/history.php?users
  - `sensorHistory`: http://php/history.php?sensors
  - `dataHistory`: http://php/history.php?data
- **Responsibilities:** Fetches historical data and transforms into LDES format

### db (MySQL)
**Role:** Relational database for the dashboard.
- **Image:** mysql:8.0
- **Internal port:** 3306
- **Initialization:** ./dashboard/database_withDeviceChannelIndex.sql
- **Data:** User data, sensor data, measurements

### php
**Role:** PHP-based web backend for the dashboard.
- **Built from:** ./dashboard/
- **Internal port:** 80
- **Mounts:** Dashboard source code, custom settings.php
- **Responsibilities:** Serves web dashboard, provides API endpoints for pipeline

### adminer
**Role:** Web-based database management UI.
- **Image:** adminer
- **Internal port:** 8080
- **Responsibilities:** Database inspection and management

## Key Features

### Dashboard Features
- **User Management:** User registration, authentication, and role-based access
- **Sensor Management:** Add, configure, and monitor various sensors
- **Data Visualization:** Real-time graphs and historical data analysis
- **Alert System:** Configurable alerts for sensor thresholds
- **CSV Import/Export:** Bulk data management capabilities
- **Multi-tenant Support:** Group-based data isolation

### Pipeline Features
- **RDF Processing:** Converts sensor data to RDF format
- **LDES Support:** Linked Data Event Streams for time-series data
- **Authentication:** WebID-based authentication for Solid compliance
- **Data Mapping:** Transforms raw sensor data to semantic formats

## Quick Start

### Prerequisites
- Docker and Docker Compose
- Git submodules support

### Installation

1. **Clone the repository with submodules:**
```bash
git clone --recursive <repository-url>
cd mumo-full
```

2. **Initialize submodules:**
```bash
git submodule update --init --recursive
```

3. **Configure settings:**
```bash
cp settings.php.example settings.php
# Edit settings.php with your configuration
```

4. **Start the application:**
```bash
docker-compose up --build
```

5. **Access the application:**
- Dashboard: http://localhost
- Adminer: http://localhost:8080

## Configuration

### Database Settings
Edit `settings.php` to configure database connection:

```php
$con = mysqli_connect("db", "user", "userpass", "mumo_test", 3306);
$url = "http://localhost"; // Base URL
$domain = "localhost";    // Domain for email addresses
$clustername = "Mumo";    // Display name
```

### Environment Variables
The pipeline uses these environment variables (configured in docker-compose.yml):

- `DEBUG`: RDF debug mode (default: rdfc)
- `groupHistory`: URL for group history API
- `userHistory`: URL for user history API  
- `sensorHistory`: URL for sensor history API
- `baseUrl`: Base URL for the application
- `dataHistory`: URL for data history API

## Production Deployment

This section guides you through deploying MuMo in a production environment with SSL encryption and a custom domain. The examples use `mumo.faro.be` as the domain - replace this with your actual domain throughout all configurations.

### Prerequisites

- Ubuntu/Debian server with Docker and Docker Compose installed
- Domain name pointing to your server's IP address
- Root or sudo access for SSL certificate installation

### Step 1: Install SSL Certificates with Certbot

1. **Install Certbot:**
```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

2. **Obtain SSL certificate** (replace with your domain):
```bash
sudo certbot --nginx -d your-domain.com
```

3. **Set up automatic renewal:**
```bash
sudo crontab -e
# Add this line for daily certificate renewal check:
0 12 * * * /usr/bin/certbot renew --quiet
```

### Step 2: Update docker-compose.yml

Modify your `docker-compose.yml` file with the following changes:

**Nginx Service - Add SSL support:**
```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
    - "443:443"        # Add SSL port
  expose:
    - "443"
    - "80"
  volumes:
    - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    # Mount SSL certificates (update paths with your domain)
    - /etc/letsencrypt/live/your-domain.com/fullchain.pem:/etc/letsencrypt/live/your-domain.com/fullchain.pem
    - /etc/letsencrypt/live/your-domain.com/privkey.pem:/etc/letsencrypt/live/your-domain.com/privkey.pem
    - /etc/letsencrypt/options-ssl-nginx.conf:/etc/letsencrypt/options-ssl-nginx.conf
    - /etc/letsencrypt/ssl-dhparams.pem:/etc/letsencrypt/ssl-dhparams.pem
  networks:
    public:
      aliases:
        - api.local
        - your-domain.com    # Add your domain
```

**Solid Service - Use HTTPS baseUrl:**
```yaml
solid:
  build: ./pipeline/
  command: /app/pipeline/auth-pipeline.ttl
  environment:
    DEBUG: rdfc
    groupHistory: http://php/history.php?groups
    userHistory: http://php/history.php?users
    sensorHistory: http://php/history.php?sensors
    baseUrl: "https://your-domain.com/"  # Important: Use HTTPS for internal requests
  volumes:
    - ./data/:/app/pipeline/ldes/
    - ./pipeline/pipeline/:/app/pipeline/
```

**Adminer Service - Remove external port exposure:**
```yaml
adminer:
  image: adminer
  expose:
    - "8080"
  # Remove ports mapping for security - only accessible internally
```

### Step 3: Update nginx.conf

Replace the entire `nginx.conf` file with this SSL-enabled configuration:

```nginx
# Docker network detection - redirects internal traffic to HTTP, external to HTTPS
map $remote_addr $from_docker {
    default         1;
    172.18.0.0/16   0;
}

server {
  listen 80;
  
  # Redirect external requests to HTTPS
  if ($from_docker) {
         return 301 https://$host$request_uri;
         break;
  }
  
  location = / {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location ~ \.php$ {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /assets {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location / {
    proxy_pass http://solid:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}

# HTTPS server block with SSL configuration
server {
  listen 443 ssl;
  # Update these paths with your domain
  ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
  include /etc/letsencrypt/options-ssl-nginx.conf;
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
  
  location = / {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location ~ \.php$ {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location /assets {
    proxy_pass http://php:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }

  location / {
    proxy_pass http://solid:3000;
    proxy_set_header    Host                $http_host;
    proxy_set_header    X-Real-IP           $remote_addr;
    proxy_set_header    X-Forwarded-Ssl     on;
    proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
    proxy_set_header    X-Forwarded-Proto   $scheme;
  }
}
```

### Step 4: Update settings.php

Modify `settings.php` to use your domain and HTTPS:

```php
<?php session_start ();
if($_SERVER['HTTP_HOST'] != "db"){
  // Production settings
  $con = mysqli_connect("db", "user", "userpass", "mumo_test", 3306);
  $url = "https://your-domain.com";     // Use HTTPS
  $domain = "your-domain.com";         // Your domain
  $clustername = "Mumo";
  $logo = "assets\images\logo\mumoLogo.png";
  $logo_w = "assets\images\logo\mumoLogoW.png";
}else{
  // Development settings (can keep local or use same domain)
  $con = mysqli_connect("db", "user", "userpass", "mumo_test", 3306);
  $url = "https://your-domain.com";     # Keep consistent with production
  $domain = "your-domain.com";
  $clustername = "Mumo";
  $logo = "assets\images\logo\Mumo final.png";
  $logo_w = "assets\images\logo\Mumo final_w.png";
}
?>
```

### Step 5: Deploy the Application

1. **Build and start the containers:**
```bash
docker-compose down
docker-compose up --build -d
```

2. **Verify the deployment:**
- Access your dashboard at: `https://your-domain.com`
- Check that HTTP redirects to HTTPS
- Verify all services are running: `docker-compose ps`

### Important Notes

**Why HTTPS in docker-compose?**
The Solid container needs to make requests to itself using the public endpoint. Since the public endpoint uses HTTPS, the `baseUrl` environment variable must use `https://` even for internal Docker communications.

**Security Considerations:**
- Change default MySQL passwords in production
- Adminer is only accessible internally (good for security)
- All external traffic is forced to use HTTPS
- SSL certificates auto-renew with the cron job

**Troubleshooting:**
- If SSL certificates aren't found, verify Certbot installation and domain ownership
- Check container logs: `docker-compose logs nginx solid php`
- Ensure your domain DNS points to the server IP before running Certbot

### Security Considerations

- **Database**: Change default MySQL passwords in production
- **SSL**: Always use HTTPS in production environments
- **Firewall**: Configure firewall to allow only necessary ports (80, 443)
- **Adminer**: Consider removing or securing database admin interface in production
- **Backups**: Implement regular database and file backups

## Development

### Local Development

For development without SSL:
1. Use the default `docker-compose.yml` configuration
2. Access via http://localhost
3. Adminer available at http://localhost:8080

### Pipeline Development

The RDF pipeline uses Turtle (.ttl) files for configuration:
- `pipeline/pipeline.ttl`: Main data processing pipeline
- `pipeline/auth-pipeline.ttl`: Authentication pipeline
- `pipeline/proc/`: Custom processor implementations

To modify pipeline behavior:
1. Edit TTL files in `pipeline/pipeline/`
2. Rebuild pipeline container: `docker-compose build pipeline`
3. Restart services: `docker-compose up -d`

### Database Schema

The database schema is defined in `dashboard/database.sql`. Key tables:
- `users`: User accounts and authentication
- `sensors`: Sensor definitions and configurations
- `measurements`: Time-series sensor data
- `groups`: User groups for multi-tenancy
- `alerts`: Alert configurations and triggers

## API Endpoints

The dashboard provides several API endpoints:
- `/history.php?users`: User history data
- `/history.php?groups`: Group history data
- `/history.php?sensors`: Sensor history data
- `/endpoint.php`: General API endpoint
- `/export.php`: Data export functionality

## Component Interactions

All services are connected to a shared Docker network named public.
This allows containers to communicate using service names as hostnames:
- http://php/...
- http://db:3306
- http://solid:3000

How Components Interact:

![Alt text](./Overview.drawio.svg)

## Troubleshooting

### Common Issues

1. **Submodule not found**: Run `git submodule update --init --recursive`
2. **Database connection failed**: Check MySQL container logs and credentials
3. **Pipeline not starting**: Verify Node.js build and dependencies
4. **SSL errors**: Ensure certificates are properly configured and paths are correct

### Logs

Check container logs for debugging:

```bash
docker-compose logs nginx      # Nginx/proxy issues
docker-compose logs php        # PHP application errors
docker-compose logs db         # Database issues
docker-compose logs solid      # Pipeline problems
```

## Commands

Start the Stack:
```bash
docker compose up --build
```

Stop the stack:
```bash
docker compose down
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with `docker-compose up --build`
5. Submit a pull request

## License

This project is licensed under the terms specified in the LICENSE file.

## Support

For support and questions:
- Check the troubleshooting section
- Review container logs
- Examine the documentation in the `dashboard/documentation/` directory
