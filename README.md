
# High-Performance C++ Cloud Drive Backend

This project is a high-concurrency cloud drive backend service built entirely from scratch in C++11. It does not rely on any third-party web frameworks (like Nginx for serving). Instead, it implements an event-driven network library from the ground up, featuring an `epoll`-based **Reactor** design pattern.

To achieve maximum performance, the system also includes a custom **thread pool**, **MySQL connection pool**, **Redis connection pool**, and a high-performance **asynchronous logging system**. The service provides a complete RESTful API for user authentication, file management, and an "instant upload" (MD5) feature, making it a lightweight and highly efficient personal storage solution.

## Core Features

  * **High-Performance Networking**: A custom network library built on `epoll` I/O multiplexing, using the **Reactor** design pattern. It employs an **I/O Thread + Worker Thread Pool** model to efficiently handle massive concurrent connections (solving the C10K problem).
  * **User Authentication**: Full support for user registration and login. User tokens (Sessions) are cached in **Redis** for fast, low-latency identity verification.
  * **Core File Operations**: Provides all essential cloud drive functions, including file upload, download, file list retrieval, file deletion, and file sharing.
  * **MD5 Instant Upload**: Implements "second upload" functionality. The client first computes the file's MD5 hash; if the hash already exists on the server, the system simply creates a user-file association in the database, skipping the actual file transfer and saving bandwidth and storage.
  * **Connection Pooling**: Includes thread-safe **MySQL Connection Pool** (`DBPool`) and **Redis Connection Pool** (`CachePool`) to reuse connections, minimize overhead, and dramatically improve database interaction performance.
  * **Asynchronous Logging**: A high-performance async logger built with a **Double-Buffering** pattern. Business threads write logs to an in-memory buffer (near-zero cost), while a dedicated logging thread handles batch flushing to disk, ensuring that logging does not block main application logic.

## Project Architecture

 \* (This is an example diagram; you can replace it as needed)\*

1.  **Client (React)**: The user interacts with the React-based frontend in their browser.
2.  **Nginx (Reverse Proxy)**: Acts as the public-facing gateway. It serves all static frontend assets (compiled React app) and reverse-proxies all API requests to the backend C++ service.
3.  **C++ Backend Service (tc\_http\_server)**:
      * **Network Layer (Reactor)**: The main I/O thread uses `epoll` to listen for all network events (new connections, readable data). It performs non-blocking network I/O only, immediately dispatching tasks.
      * **HTTP Parsing Layer**: Uses `http-parser` (from Nginx) to efficiently parse raw HTTP requests. The `ApiCommon` module then routes requests based on the URL path.
      * **Business Logic Layer**: All business logic (e.g., login, upload) is encapsulated.
      * **Thread Pool**: CPU-bound or blocking tasks (like database queries) are pushed into a task queue and executed by a pool of worker threads, completely separating I/O from computation.
4.  **Data Layer**:
      * **Redis (Cache)**: Managed by `CachePool`, provides high-speed storage and retrieval for user login tokens.
      * **MySQL (Database)**: Managed by `DBPool`, provides persistent storage for user info, file metadata, and user-file relationships.

## Tech Stack

  * **Backend Language**: C++11
  * **Network Model**:
      * Custom Network Library (based on `epoll` I/O Multiplexing)
      * Reactor Event-Driven Model
      * Multithreading (`std::thread`, `pthread_mutex_t`)
  * **Core Custom Components**:
      * `ThreadPool` (Worker Thread Pool)
      * `DBPool` (MySQL Connection Pool)
      * `CachePool` (Redis Connection Pool)
      * `AsyncLogging` (Asynchronous Logging System)
  * **Databases**:
      * MySQL (Primary database)
      * Redis (Caching user tokens)
  * **Third-Party Libraries**:
      * `libhiredis` (Redis C client)
      * `libmysqlclient` (MySQL C client)
      * `jsoncpp` (Parsing/generating JSON)
      * `http-parser` (Nginx's open-source HTTP parser)
  * **Frontend**: React.js
  * **Build & Deployment**:
      * CMake (Backend build system)
  * Nginx (Reverse proxy and static file server)

## Quick Start

### 1\. Prerequisites

  * **OS**: Linux (required for `epoll`)
  * **Databases**: MySQL (e.g., 5.7+), Redis
  * **Compiler**: G++ (with C++11 support)
  * **Build Tools**: CMake, Make
  * **Web Server**: Nginx
  * **Frontend**: Node.js, Yarn (to build the React UI)

### 2\. Database & Cache Setup

1.  Start your MySQL and Redis services.

2.  Ensure Redis is running (default port 6379).

3.  Import the `tuchuang.sql` file (located in the project root) into MySQL to create the necessary database (`tuchuang`) and tables.

    ```bash
    # Log in to MySQL
    mysql -u root -p

    # Create the database
    mysql> CREATE DATABASE tuchuang;

    # Import the tables
    mysql> USE tuchuang;
    mysql> SOURCE /path/to/your/project/tuchuang.sql;
    ```

### 3\. Backend Compilation &- Run (tc\_http\_server)

1.  Navigate to the project root, then create and enter a `build` directory.
    ```bash
    mkdir build
    cd build
    ```
2.  Use CMake to generate the Makefile.
    ```bash
    cmake ..
    ```
3.  Compile the project.
    ```bash
    make
    ```
4.  An executable named `tc_http_server` will be created in the `build` directory.
5.  Copy the configuration file into the `build` directory.
    ```bash
    cp ../tc_http_server.conf .
    ```
6.  **Important**: Edit `tc_http_server.conf` and update the IP, port, username, and password for your MySQL and Redis instances.
7.  Run the backend server.
    ```bash
    ./tc_http_server
    ```
    *Tip: If you encounter MySQL errors on startup, double-check your `tc_http_server.conf` settings and ensure the `tuchuang.sql` script was imported successfully.*

### 4\. Frontend Build & Nginx Deployment

This project uses a separate frontend/backend deployment.

1.  **Build Frontend**:
    (Assuming you have the React frontend code in a `tc-front` directory)

    ```bash
    cd /path/to/your/tc-front
    yarn install  # Install dependencies
    yarn build    # Create production build
    ```

    This will create a `build` folder inside `tc-front` containing the static HTML, CSS, and JS files.

2.  **Configure Nginx**:
    Copy all files from the frontend `build` directory to your Nginx web root (e.g., `/usr/local/nginx/html` or `/var/www/html`).

3.  Edit your Nginx configuration file (e.g., `/usr/local/nginx/conf/nginx.conf`) to serve the frontend and proxy API calls to the C++ backend.

    Here is a sample configuration:

    ```nginx
    server {
        listen       80;
        server_name  localhost; # Replace with your domain or IP
        
        # Root directory for frontend static files
        # root /path/to/your/frontend/build;
        root /usr/local/nginx/html; # Example path
        index index.html index.htm;
        
        # Handle React-Router history mode (prevents 404s on page refresh)
        location / {
            try_files $uri $uri/ /index.html;
        }

        # API Reverse Proxy
        # Forward all requests starting with /api/ to the C++ backend
        # Assumes tc_http_server is running on 127.0.0.1:8888
        location ^~ /api/ {
            proxy_pass http://127.0.0.1:8888; # Match the port in tc_http_server.conf
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # Error pages
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    ```

4.  Reload your Nginx service.

    ```bash
    sudo nginx -s reload
    ```

5.  Visit `http://localhost` (or your server IP) in your browser to use the application.

