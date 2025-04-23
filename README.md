# Nginx configuration for dockerized django app

It's always pain in the a$$ to set up nginx and to get it working properly.

This is what works for me!

## Docker Compose Setup

```yaml
# docker-compose.yml


services:
  # for renewing SSL certificates
  certbot:
    container_name: certbot
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    depends_on:
      - nginx

  # Nginx (Reverse Proxy)
  nginx:
    container_name: nginx
    image: nginx:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro  # Mount Nginx config
      - ./certbot/conf:/etc/letsencrypt  # Mount SSL certificates
      - ./certbot/www:/var/www/certbot  # Certbot verification folder
      - ./your_django_project/staticfiles:/app/staticfiles
    depends_on:
      - django-app
    networks:
      - django-network

  # UI
  django-app:
    container_name: django-app
    build:
      context: .
      dockerfile: Dockerfile
    restart: always
    environment:
      - SERVICE_NAME=django-app
      - PYTHONUNBUFFERED=1
      - DJANGO_ENV=prod
      - DJANGO_SETTINGS_MODULE=your_django_project.settings
    volumes:
      - .:/app
    # expose port 8000 only on docker network
    expose:
      - "8000"
    entrypoint: >
      /bin/sh -c "python manage.py collectstatic --noinput &&
      python -m gunicorn your_django_project.asgi:application -k uvicorn_worker.UvicornWorker --bind 0.0.0.0:8000"
    networks:
      - django-network
  

  # no DB, assuming you are using SQLite


networks:
  django-network:
    name: django-network
    driver: bridge


```


## Nginx Configuration

```conf
# ./nginx/nginx.conf

# great exemplary configuration for nginx
# https://gist.github.com/duduribeiro/9508966
# great site for nginx troubleshooting
# https://redbot.org/

events {}

http {

    ##
    # MIME-TYPE
    ##

    include /etc/nginx/mime.types;
    # fallback in case we can't determine a type
    # if not set then response will be set to text/plain
    default_type  application/octet-stream;

    ##
    # GZIP
    ##

    # don't need to use djangos GzipMiddleware, nginx is faster and designed for this

    # In production you MUST set gzip to "on" in order to save bandwidth. Web browsers
    # which handle compressed files (all recent ones do) will get a very smaller version
    # of the server response. 
    gzip  on;
    # Enables compression for a given HTTP request version.
    gzip_http_version 1.0;
    # Compression level 1 (fastest) to 9 (slowest).
    gzip_comp_level 6;
    # Enables compression for all proxied requests.
    gzip_proxied any;
    # Minimum length of the response (bytes). Responses shorter than this length will not be compressed.
    gzip_min_length 10000;
    # Enables compression for additional MIME-types.
    gzip_types  text/plain text/css application/x-javascript text/xml
                application/xml application/xml+rss text/javascript application/json application/javascript;
    # Disables gzip compression for User-Agents matching the given regular expression.
    # Is this case we've disabled gzip for old versions of the IE that don't support compressed responses.
    gzip_disable "MSIE [1-6] \.";

    # add the Vary header to the response
    # pointed out by redbot.org
    gzip_vary on;

    
    upstream django {
        # container name from docker-compose.yml
        server django-app:8000;
    }

    server {
        listen 80;

        server_name yourdomain.com www.yourdomain.com;

        # for ssl certificate creation
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
            try_files $uri =404;
        }

        # redirect http traffic to https
        location / {
            return 301 https://$host$request_uri;
        }
    }

    # Secure HTTPS server
    server {
        listen 443 ssl;
        server_name yourdomain.com www.yourdomain.com;

        ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

        location / {
            proxy_pass http://django;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # value of STATIC_URL from django settings
        location /static/ {
            # value of STATIC_ROOT from django settings
            # mounted to docker volume
            alias /app/staticfiles/;

            # Enable browser caching
            # I am adding quite and aggressive caching for static files
            # because I am using ManifestStaticFilesStorage - which generates hashes
            # at the end of the file name (depending on the content of the file)
            # https://docs.djangoproject.com/en/5.2/ref/contrib/staticfiles/#manifeststaticfilesstorage
            expires 1y;                         # Set expires header for 1 year
            add_header Cache-Control "public";  # Allow public caching


            # this is for CDNs and external caching systems
            # This configuration will make Nginx correctly respond with 304 Not Modified
            # when clients send conditional requests with matching ETags or If-Modified-Since headers
            # Enable conditional requests handling
            etag on;
            if_modified_since exact;

            # seems redundant since we have gzip_vary on
            # but it ensure this header is included in ALL response
            # including 304 responses
            add_header Vary Accept-Encoding always;
            
            # Required for proper cache validation
            add_header Cache-Control "must-revalidate";
        }

        # media files should have similar config, but I have mine on cloud


        # Defines the static page for HTTP status 404
        # TODO: need to make error templates into static files, so that nginx can serve them
        #error_page 404 /path/to/404.html;

        # Defines the static page for HTTP status 500
        # TODO: need to make error templates into static files, so that nginx can serve them
        #error_page 500 /path/to/500.html;
    }
}
```

## Make Commands

```makefile
# Makefile

# Let's encrypt expiry bot will send you email
# when ssl is close to expiring
email=youremail.com
domain=yourdomain.com

run:
	docker compose up

run_silent:
	docker compose up -d

stop:
	docker compose down

# before requesting a certificate, run this test
# to validate that everything is set up correctly
ssl_test:
    docker compose run --rm certbot certonly --webroot -w /var/www/certbot --email ${email} --agree-tos --no-eff-email --force-renewal -d ${domain} -d www.${domain} --dry-run

# request/renew SSL certificate, can also make it into a cron job
ssl_request:
    docker compose run --rm certbot certonly --webroot -w /var/www/certbot --email ${email} --agree-tos --no-eff-email --force-renewal -d ${domain} -d www.${domain}

# will reload nginx config without
# any downtime, (if you don't have millions of users,
# this is just pure mental ma$turabtion, but here we are :)
# https://nginx.org/en/docs/beginners_guide.html#control
hot_reload_nginx_conf:
	docker compose exec nginx nginx -s reload
```