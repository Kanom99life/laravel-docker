# Laravel + PostgreSQL + Docker Setup

This guide shows how to run a Laravel application with a PostgreSQL database using Docker and Docker Compose.

---

## 📦 1. PostgreSQL Setup with Docker Compose

Create a `docker-compose.yml` file:

```yaml
services:
  postgres:
    image: postgres:15  # Use PostgreSQL version 15 image
    container_name: laravel_postgres  # Custom container name
    environment:
      POSTGRES_DB: my_laravel_db  # Name of the default database to create
      POSTGRES_USER: user  # Username for the database
      POSTGRES_PASSWORD: password  # Password for the database user
    ports:
      - "5432:5432"  # Map port 5432 (Postgres default) to host
    volumes:
      - pgdata:/var/lib/postgresql/data  # Persist database data in a named volume

# Named Volumes
volumes:
  pgdata:  # Declare the named volume used by Postgres
```


## ⚙️ 2. Laravel Environment Configuration

Edit your .env file in Laravel:

```
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=my_laravel_db
DB_USERNAME=user
DB_PASSWORD=password
```

## 🐘 3. Dockerfile for Laravel (PHP-FPM)

Create a Dockerfile:

```dockerfile
# Use the official PHP 8.2 FPM image as the base
FROM php:8.2-fpm

# Set the working directory inside the container
WORKDIR /var/www

# Install system-level dependencies needed for Laravel and PHP extensions
RUN apt-get update && apt-get install -y \
    build-essential \            # For compiling PHP extensions and dependencies
    libpng-dev \                 # Required for image manipulation (GD)
    libjpeg-dev \                # JPEG support for GD
    libonig-dev \                # Required for multibyte string functions (mbstring)
    libxml2-dev \                # Required for XML parsing and related PHP extensions
    zip unzip \                  # For handling zip files (e.g., composer install)
    curl \                       # Required for downloading tools, APIs
    git \                        # Useful for Composer when installing packages from git
    libpq-dev \                  # PostgreSQL client libraries for pdo_pgsql
    && docker-php-ext-install \  # Install PHP extensions:
        pdo \                    # PDO core
        pdo_pgsql \              # PostgreSQL support via PDO
        mbstring \               # Multibyte string handling (needed by Laravel)
        exif \                   # For image metadata
        pcntl \                  # For managing processes (queues, etc.)
        bcmath \                 # Arbitrary precision math (needed for Laravel)
        gd                       # Image processing (requires libpng/libjpeg)

# Install Composer from the official Composer image using multi-stage copy
COPY --from=composer/composer:latest-bin /composer /usr/bin/composer

# Copy the entire Laravel application into the container
COPY . .

# Install Laravel dependencies with Composer
# --no-dev: Skip development packages (for production)
# --optimize-autoloader: Optimizes PSR autoloading for better performance
RUN composer install --no-dev --optimize-autoloader

# Set proper permissions so that Laravel can write to storage and cache
RUN chown -R www-data:www-data /var/www \
    && chmod -R 755 /var/www/storage

# Expose port 9000 (used by PHP-FPM to communicate with Nginx)
EXPOSE 9000

# Start the PHP-FPM server
CMD ["php-fpm"]

```

## 🌐 4. Full Stack Docker Compose (Laravel + PostgreSQL + Nginx)

```yaml
services:
  # Backend Application Service (e.g., Laravel with PHP-FPM)
  app:
    build: .  # Build the Docker image using the Dockerfile in the current directory
    container_name: laravel_app  # Name the container for easier reference
    volumes:
      - .:/var/www  # Mount the current project directory into the container at /var/www
    depends_on:
      - postgres  # Ensure the database service is started before the app

  # Nginx Web Server Service
  nginx:
    image: nginx:alpine  # Use a lightweight Nginx image based on Alpine Linux
    container_name: laravel_nginx  # Custom container name
    ports:
      - "80:80"  # Map port 80 on the host to port 80 in the container
    volumes:
      - .:/var/www  # Mount the app code so Nginx can serve static files (and .php requests)
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf  # Use a custom Nginx config file
    depends_on:
      - app  # Ensure the app container starts before Nginx

  # PostgreSQL Database Service
  postgres:
    image: postgres:15  # Use PostgreSQL version 15 image
    container_name: laravel_postgres  # Custom container name
    environment:
      POSTGRES_DB: my_laravel_db  # Name of the default database to create
      POSTGRES_USER: user  # Username for the database
      POSTGRES_PASSWORD: password  # Password for the database user
    ports:
      - "5432:5432"  # Map port 5432 (Postgres default) to host
    volumes:
      - pgdata:/var/lib/postgresql/data  # Persist database data in a named volume

# Named Volumes
volumes:
  pgdata:  # Declare the named volume used by Postgres

```

## 📝 5. Nginx Configuration

Create nginx/default.conf:

```nginx
server {
    # Listen for incoming HTTP requests on port 80 (default HTTP port)
    listen 80;

    # Specify the default index files to look for when a directory is requested
    index index.php index.html;

    # Server name (hostname) that this config applies to. Here it's just 'localhost'.
    server_name localhost;

    # Define the root directory for serving files
    root /var/www/public;

    # This block handles all general requests
    location / {
        # Try to serve the file directly, or the directory, 
        # or fallback to index.php passing the original query string.
        try_files $uri $uri/ /index.php?$query_string;
    }

    # This block matches any request that ends with '.php'
    location ~ \.php$ {
        # Load default fastcgi parameters (e.g., SCRIPT_NAME, REQUEST_METHOD, etc.)
        include fastcgi_params;

        # Forward PHP requests to a backend service listening on port 9000.
        # 'app' refers to the container name in Docker Compose.
        fastcgi_pass app:9000;

        # Default PHP file to execute if the script name is a directory
        fastcgi_index index.php;

        # Set the full path to the script to execute
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;

        # Set PATH_INFO to allow PHP to interpret URL path parts correctly
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    # Deny access to any files starting with .ht (e.g., .htaccess)
    location ~ /\.ht {
        deny all;
    }
}

```

## 🚀 6. Starting the Project

Start the containers:

```bash
docker-compose up --build -d
```
Run migrations:
```bash
docker exec -it laravel_app php artisan migrate
```

## 🛠7. Common Errors & Fixes

❌ Laravel can't connect to PostgreSQL
```bash
SQLSTATE[08006] [7] connection to server at "127.0.0.1", port 5432 failed: Connection refused
```
Laravel inside the container sees 127.0.0.1 as itself, not your PostgreSQL container.

So this will not work inside Docker.
```bash
DB_HOST=127.0.0.1
```

✅ Fix: Laravel is inside a container. Change DB_HOST in .env to the PostgreSQL service name:

```env
DB_HOST=postgres
```