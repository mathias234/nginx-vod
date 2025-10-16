# nginx with nginx-vod-module Docker Container

A Docker container with nginx and nginx-vod-module compiled from source, based on Ubuntu 24.04 LTS.

## Versions

- **Ubuntu**: 24.04 LTS (Noble Numbat)
- **nginx**: 1.28.0 (stable)
- **nginx-vod-module**: 1.33

## Features

- Multi-stage build for minimal image size
- nginx compiled from source with nginx-vod-module statically linked
- Based on latest Ubuntu LTS
- All standard nginx modules included
- HTTP/2 support
- SSL/TLS support
- Runs as non-root user (nginx)

## Quick Start

### Building the Image

```bash
docker build -t nginx-vod:latest .
```

### Running with Docker

```bash
docker run -d \
  -p 80:80 \
  -p 443:443 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/videos:/var/www/videos:ro \
  --name nginx-vod \
  nginx-vod:latest
```

### Running with Docker Compose

```bash
docker-compose up -d
```

## Configuration

The container expects you to mount your own nginx configuration file. Create an `nginx.conf` file in your project directory.

### Example nginx.conf for VOD Streaming

```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    server {
        listen 80;
        server_name localhost;

        # VOD settings
        vod_mode local;
        vod_metadata_cache metadata_cache 512m;
        vod_response_cache response_cache 128m;
        vod_last_modified_types *;
        vod_segment_duration 9000;
        vod_align_segments_to_key_frames on;
        vod_dash_fragment_file_name_prefix "segment";
        vod_hls_segment_file_name_prefix "segment";

        vod_manifest_segment_durations_mode accurate;

        open_file_cache max=1000 inactive=5m;
        open_file_cache_valid 2m;
        open_file_cache_min_uses 1;
        open_file_cache_errors on;

        location /hls/ {
            vod hls;
            alias /var/www/videos/;
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Allow-Origin '*';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
        }

        location /dash/ {
            vod dash;
            alias /var/www/videos/;
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Allow-Origin '*';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
        }

        location /thumb/ {
            vod thumb;
            alias /var/www/videos/;
            add_header Access-Control-Allow-Headers '*';
            add_header Access-Control-Allow-Origin '*';
            add_header Access-Control-Allow-Methods 'GET, HEAD, OPTIONS';
        }
    }
}
```

## Volume Mounts

### Required

- `/etc/nginx/nginx.conf` - Your nginx configuration file

### Optional

- `/var/www/videos` - Directory containing your video files
- `/etc/nginx/conf.d` - Additional configuration files
- `/etc/nginx/certs` - SSL certificates
- `/var/log/nginx` - nginx logs (already forwarded to stdout/stderr)

## Build Arguments

You can customize the versions during build:

```bash
docker build \
  --build-arg NGINX_VERSION=1.28.0 \
  --build-arg VOD_MODULE_VERSION=1.33 \
  -t nginx-vod:latest .
```

## Verifying the Installation

Check if nginx-vod-module is loaded:

```bash
docker exec nginx-vod nginx -V
```

You should see `--add-module=/tmp/nginx-vod-module` in the configure arguments.

## Testing

Test your configuration before starting:

```bash
docker run --rm \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx-vod:latest \
  nginx -t
```

## References

- [nginx-vod-module GitHub](https://github.com/kaltura/nginx-vod-module)
- [nginx Documentation](https://nginx.org/en/docs/)
- [nginx-vod-module Documentation](https://github.com/kaltura/nginx-vod-module/blob/master/README.md)
