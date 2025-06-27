
# Home Lab Operations Runbook

This document is the central source of truth for the architecture, configuration, and operation of this home lab. Its purpose is to ensure clear, repeatable processes and to serve as a log for future incident management.

## 1\. System Mind Map

    mindmap
      root((Home Lab))
        Services
          (@1)Container Registry
            Domain: registry.tirro.ir
            Technology: Docker Registry
            Storage: /var/registry/data
        Infrastructure
          Kubernetes
            Distribution: MicroK8s
            CNI: Calico
    

## 2\. Container Registry

This section details the setup and maintenance of the self-hosted container registry.

### 2.1. Architectural Decision

-   **Decision:** We will deploy a self-hosted Docker container registry (`registry:2`). It will be configured to be secure (TLS and user authentication), act as a pull-through cache for public registries (like Docker Hub), and use an outbound proxy to fetch images. This solves the initial problem of `403 Forbidden` errors on CNI image pulls.
    

### 2.2. Pros & Cons

-   **Pros:**
    
    -   **Secure:** All traffic is encrypted via TLS, and access is controlled by user credentials.
        
    -   **Solves Geo-Blocking:** Outgoing image requests are routed through a specified proxy, bypassing regional blocks.
        
    -   **Enables CI/CD:** Provides a private, reliable, and secure target for pushing custom application images.
        
    -   **Improves Performance & Reliability:** Caches external images on the local network, speeding up deployments and reducing dependency on internet stability.
        
-   **Cons:**
    
    -   **Initial Setup Overhead:** Requires configuration of DNS, firewall, TLS certificates, and user credentials.
        
    -   **Single Point of Failure:** The lab's ability to pull images depends on this single container. (This is an acceptable risk for a home lab environment).
        

### 2.3. Alternatives Considered

-   **Manual Image Loading:** Loading images via `docker save`/`docker load` or `microk8s ctr image import`. Rejected due to being a manual, non-scalable process that creates operational toil.
    
-   **System-Wide Proxy:** Configuring a proxy for the entire container runtime. Rejected in favor of the registry approach, which provides more benefits (caching, private storage) for similar effort.
    

### 2.4. Core Configuration (`config.yml`)

This is the primary configuration file for the registry service. It should be created at `/opt/registry/config.yml` on the host machine.

    version: 0.1
    # --- LOGGING ---
    # Enables detailed, machine-readable logging for easier troubleshooting.
    log:
      level: debug
      formatter: json
      fields:
        service: registry
    
    # --- STORAGE ---
    # Configures local filesystem storage and enables image deletion via the API.
    storage:
      filesystem:
        rootdirectory: /var/lib/registry
      delete:
        enabled: true
    
    # --- AUTHENTICATION ---
    # Enforces user authentication for all API requests.
    auth:
      htpasswd:
        realm: local-registry
        path: /etc/docker/registry/auth/htpasswd
    
    # --- HTTP & TLS ---
    # Configures the registry to serve traffic over HTTPS using our domain and TLS certs.
    http:
      addr: :5000
      host: [https://registry.tirro.ir](https://registry.tirro.ir)
      tls:
        certificate: /etc/docker/registry/tls/fullchain.pem
        key: /etc/docker/registry/tls/privkey.pem
      accesslog:
        disabled: false
    
    # --- PULL-THROUGH CACHE ---
    # Configures the registry to act as a cache for Docker Hub.
    # The proxy it USES to fetch these images is set by environment variables, NOT here.
    proxy:
      remoteurl: [https://registry-1.docker.io](https://registry-1.docker.io)
    
    # --- HEALTH ---
    # Enables a health check endpoint for the storage driver.
    health:
      storagedriver:
        enabled: true
        interval: 10s
        threshold: 3
    

### 2.5. Setup & Implementation

Follow these steps to deploy the registry.

#### **Step 1: Prepare Host Directories**

Create the necessary directories on the host machine. The registry container will mount these to persist data and load configuration.

    # Create directory for registry data on the separate partition mount
    sudo mkdir -p /var/registry/data
    
    # Create directories for configuration, TLS certs, and auth files
    sudo mkdir -p /opt/registry/tls
    sudo mkdir -p /opt/registry/auth
    
    # Create the empty config and auth files
    sudo touch /opt/registry/config.yml
    sudo touch /opt/registry/auth/htpasswd
    

#### **Step 2: Populate `config.yml`**

Copy the YAML content from section **2.4** into `/opt/registry/config.yml`.

#### **Step** 3: Generate **TLS Certificates (Let's Encrypt)**

1.  **DNS:** Ensure `registry.tirro.ir` has an A record pointing to your static IP.
    
2.  **Firewall:** Temporarily open port 80 on your firewall and forward it to your lab machine.
    
3.  **Certbot:** Install and run `certbot` to issue the certificate.
    
        sudo snap install --classic certbot
        sudo certbot certonly --standalone -d registry.tirro.ir
        
    
4.  **Copy Certs:** Copy the generated certificates to the volume mount location.
    
        CERT_PATH="/etc/letsencrypt/live/registry.tirro.ir"
        sudo cp "${CERT_PATH}/fullchain.pem" /opt/registry/tls/
        sudo cp "${CERT_PATH}/privkey.pem" /opt/registry/tls/
        
    
5.  **Firewall:** Close port 80.
    

#### **Step 4: Create User Credentials**

Use `htpasswd` to create a username and password. You will be prompted to enter a password.

    # Install apache2-utils if not already present
    sudo apt-get update && sudo apt-get install -y apache2-utils
    
    # Create the first user (e.g., 'ci-user')
    sudo htpasswd -cB /opt/registry/auth/htpasswd ci-user
    

#### **Step** 5: Run the **Registry Container**

This command starts the registry, mounts all the necessary files, and critically, passes the outbound proxy information as environment variables.

    # --- IMPORTANT ---
    # Set this to the proxy that allows you to bypass geo-restrictions
    export OUTBOUND_PROXY="[http://your-outbound-proxy.com:8080](http://your-outbound-proxy.com:8080)"
    
    # Run the container
    docker run -d \
      -p 5000:5000 \
      --restart=always \
      --name registry \
      -v /var/registry/data:/var/lib/registry \
      -v /opt/registry/config.yml:/etc/docker/registry/config.yml:ro \
      -v /opt/registry/tls:/etc/docker/registry/tls:ro \
      -v /opt/registry/auth:/etc/docker/registry/auth:ro \
      -e "HTTP_PROXY=${OUTBOUND_PROXY}" \
      -e "HTTPS_PROXY=${OUTBOUND_PROXY}" \
      registry:2
    

### 2.6. Troubleshooting Guide

-   **Check Logs:** `docker logs registry` (Logs are in JSON format).
    
-   **Authentication Failure:** `curl -u "user:pass" https://registry.tirro.ir:5000/v2/_catalog`. A `401 Unauthorized` response means credentials are bad. Check the `htpasswd` file.
    
-   **TLS Error:** An SSL/TLS error from `curl` likely means the certificates weren't copied correctly, the paths in `config.yml` are wrong, or the container wasn't restarted after a change.
    
-   **500 Internal Server Error:** Often points to a storage or configuration problem. Check the logs for specific error messages




