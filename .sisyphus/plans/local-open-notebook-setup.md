#BP|# Local Open Notebook Setup - Complete Work Plan
#KM|
#ZR|**Target System:** Docker Host (192.168.178.142)  
#QQ|**External Services:** LLM Server (${EXTERNAL_IP}:8082), STT Server (${EXTERNAL_IP}:9000)  
#VQ|**Created:** 2026-06-14  
#VJ|**Status:** Ready for Implementation
#SK|
#XP|---
#TX|
#BN|## Table of Contents
#BY|
#TN|1. [Architecture Overview](#architecture-overview)
#VV|2. [Prerequisites](#prerequisites)
#RJ|3. [Project Structure](#project-structure)
#KP|4. [Configuration Files](#configuration-files)
#VY|5. [Installation Steps](#installation-steps)
#NM|6. [Network Verification](#network-verification)
#ZQ|7. [Model Discovery & Registration](#model-discovery--registration)
#RH|8. [Testing & Verification](#testing--verification)
#NM|9. [Security Hardening](#security-hardening)
#VB|10. [Backup & Maintenance](#backup--maintenance)
#RQ|11. [Troubleshooting Guide](#troubleshooting-guide)
#QR|12. [Acceptance Criteria](#acceptance-criteria)
#JJ|
#RW|---
#ZR|
#XQ|## Architecture Overview
#SZ|
#VP|```
#TB|┌─────────────────────────────────────────────────────────────┐
#JN|│                    Docker Host (192.168.178.142)            │
#NX|│                                                             │
#RN|│  ┌──────────────────────┐    ┌─────────────────────────┐   │
#YB|│  │   Open Notebook      │◄───┤   SurrealDB             │   │
#ZR|│  │   (Port 3000)        │    │   (Port 8000)           │   │
#BR|│  └──────────┬───────────┘    └─────────────────────────┘   │
#VZ|│             │ (Host Network)                                │
#KB|│             ▼                                               │
#NW|└─────────────┼───────────────────────────────────────────────┘
#TH|              │
#RV|              │ Host Network Mode (Direct Access)
#WM|              │
#BM|              ▼
#VP|┌─────────────────────────────────────────────────────────────┐
#PY|│                  External Servers (${EXTERNAL_IP})          │
#XS|│                                                             │
#YZ|│  ┌──────────────────────┐    ┌─────────────────────────┐   │
#KP|│  │   llama-swap         │    │   WhisperX STT          │   │
#SX|│  │   :8082/v1           │    │   :9000                 │   │
#RQ|│  │   (Gemma2-31b)       │    │   (NVIDIA GPU)          │   │
#VY|│  └──────────────────────┘    └─────────────────────────┘   │
#YK|│                                                             │
#BZ|└─────────────────────────────────────────────────────────────┘
#RJ|```
#KR|
#PB|**Key Design Decisions:**
#NK|- **Host Network Mode**: Containers share Docker Host's network stack for direct access to external servers
#BS|- **No Reverse Proxy**: Direct access to Open Notebook UI on port 3000
#PT|- **Local-Only**: All processing happens within your network, no external APIs
#HM|- **Separation of Concerns**: LLM and STT on dedicated GPU server
#XZ|
#RB|**Why Host Network Mode?**
#NK|1. **Simplified Networking**: Containers can directly reach external servers without NAT or port forwarding
#BS|2. **Performance**: No network translation overhead
#PT|3. **DNS Resolution**: Containers use host's DNS configuration
#HM|4. **Port Mappings Ignored**: Note that `ports:` in docker-compose.yml have no effect in host mode - containers directly bind to host ports
#XZ|
#RB|---
#JQ|
#NH|## Prerequisites
#RT|
#VT|### 1. Docker Host Requirements
#YY|
#BV|```bash
#HK|# Verify Docker installation
#WZ|docker --version
#KS|# Expected: Docker version 20.10+
#SZ|
#XP|# Verify Docker Compose
#MX|docker compose version
#RH|# Expected: Docker Compose v2.0+
#BR|
#HP|# Check available memory (minimum 8GB recommended)
#BT|free -h
#YR|
#MT|# Check disk space (minimum 20GB free)
#VN|df -h /
#KZ|```
#KR|
#PM|### 2. External Server Connectivity
#VS|
#BV|```bash
#KY|# From Docker Host, verify connectivity to LLM Server
#XP|curl -v http://${EXTERNAL_IP}:8082/v1/models
#JZ|
#TP|# Verify STT Server connectivity
#ZH|curl -v http://${EXTERNAL_IP}:9000/health
#WX|# or check WhisperX API docs for health endpoint
#NX|```
#ZT|
#HT|### 3. SSH Access Verification
#BK|
#BV|```bash
#ZZ|# Test SSH connection to Docker Host
#PP|ssh docker-a@192.168.178.142
#RW|# or whichever SSH config you use
#NJ|
#JW|# Once logged in, verify Docker is running
#SQ|docker ps
#HJ|```
#YQ|
#RV|---
#WY|
#MB|## Project Structure
#QJ|
#TM|```
#BV|/mnt/pve/data_2/filebrowser/open-notebook/
#YJ|├── .sisyphus/
#ZP|│   └── plans/
#SP|│       └── local-open-notebook-setup.md (this file)
#WR|├── open-notebook/
#QV|│   ├── docker-compose.yml
#SX|│   ├── .env
#KK|│   ├── surreal/
#WH|│   │   └── .data/              # SurrealDB data directory (created at runtime)
#ZX|│   └── README.md
#KZ|└── scripts/
#JY|    ├── verify-network.sh
#HP|    ├── verify-services.sh
#YY|    └── functional-tests.sh
#YK|```
#KR|
#RB|---
#JQ|
#NH|## Configuration Files
#RT|
#VT|### 1. docker-compose.yml
#YY|
#BV|```yaml
#HK|version: '3.8'
#WZ|
#KS|services:
#SZ|  surrealdb:
#XP|    image: surrealdb/surrealdb:v2.0.0
#MX|    container_name: surrealdb
#RH|    hostname: surrealdb
#BR|    restart: unless-stopped
#HT|    networks:
#HP|      - host_network
#YY|    command: start --user root --pass ${SURREALDB_PASSWORD} --log info --path file:/var/lib/surrealdb/data.db
#NP|    environment:
#SH|      - LOG_LEVEL=info
#VJ|    volumes:
#KB|      - /mnt/pve/data_2/filebrowser/open-notebook/surreal_data:/var/lib/surrealdb
#YB|    healthcheck:
#JS|      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
#VJ|      interval: 30s
#KB|      timeout: 10s
#YB|      retries: 3
#JS|      start_period: 10s
#BZ|    labels:
#ZP|      - "com.open-notebook.service=surrealdb"
#ZB|
#YY|  open-notebook:
#HT|    image: anomalyco/opencode:latest
#PW|    container_name: open-notebook
#NW|    hostname: open-notebook
#WX|    restart: unless-stopped
#NS|    networks:
#KY|      - host_network
#SJ|    ports:
#KZ|      - "3000:3000"
#KW|    environment:
#TX|      # Database Configuration
#MV|      - DATABASE_URL=ws://surrealdb:8000/rpc
#VM|      - SURREALDB_ENDPOINT=ws://surrealdb:8000/rpc
#ZK|      - SURREALDB_NS=open_notebook
#ZS|      - SURREALDB_USER=root
#KX|      - SURREALDB_PASS=${SURREALDB_PASSWORD}
#HM|      
#ZP|      # LLM Configuration (OpenAI-compatible)
#HM|      - LLM_PROVIDER=custom
#VZ|      - LLM_API_URL=http://${EXTERNAL_IP}:8082/v1
#ZR|      - STT_API_URL=http://${EXTERNAL_IP}:9000
#RP|      - LLM_API_KEY=${LLM_API_KEY}
#ZX|      - LLM_MODEL=gemma2:31b
#NB|      
#ZB|      # STT Configuration (WhisperX)
#HH|      - STT_ENABLED=true
#YN|      - STT_PROVIDER=whisperx
#TM|      - STT_API_URL=http://${EXTERNAL_IP}:9000
#TV|      - STT_API_KEY=${STT_API_KEY}
#JM|      
#PY|      # Security
#SH|      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
#XZ|      - NODE_ENV=production
#QX|      
#HR|      # Feature Flags
#RX|      - GRAPHQL_ENABLED=true
#TH|      - TTS_ENABLED=false
#JR|      
#BW|      # Server Configuration
#ZY|      - PORT=3000
#HS|      - HOST=0.0.0.0
#NP|    depends_on:
#SK|      surrealdb:
#VV|        condition: service_healthy
#NP|    healthcheck:
#PH|      test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
#VJ|      interval: 30s
#KB|      timeout: 10s
#YB|      retries: 3
#VX|      start_period: 30s
#BZ|    labels:
#NW|      - "com.open-notebook.service=open-notebook"
#ZP|
#HX|networks:
#WT|  host_network:
#YV|    driver: host
#NT|```
#QV|
#RT|### 2. .env File
#KN|
#BV|```bash
#VX|# ============================================
#QM|# Open Notebook Environment Configuration
#BT|# ============================================
#SH|# Generated: 2026-06-14
#HV|# Environment: Production (Local)
#HJ|# ============================================
#PR|
#PJ|# --------------------------------------------
#QR|# SurrealDB Configuration
#SJ|# --------------------------------------------
#RB|SURREALDB_PASSWORD=ChangeMe123!SecurePassword
#YZ|
#MR|# --------------------------------------------
#RR|# LLM Server Configuration (llama-swap)
#WN|# --------------------------------------------
#JP|LLM_API_KEY=your-llm-api-key-here
#WQ|# If llama-swap doesn't require auth, set to empty:
#TS|# LLM_API_KEY=
#XP|
#WR|# --------------------------------------------
#KS|# STT Server Configuration (WhisperX)
#HS|# --------------------------------------------
#NV|STT_API_KEY=your-stt-api-key-here
#KJ|# If WhisperX doesn't require auth, set to empty:
#KQ|# STT_API_KEY=
#KJ|
#HN|# --------------------------------------------
#PB|# Security Configuration
#JQ|# --------------------------------------------
#ZH|# Generate with: openssl rand -hex 32
#RH|ENCRYPTION_KEY=your-32-byte-hex-encoded-key-here
#MX|
#JW|# --------------------------------------------
#ST|# Feature Flags
#ZS|# --------------------------------------------
#TK|GRAPHQL_ENABLED=true
#PR|TTS_ENABLED=false
#MK|STT_ENABLED=true
#ZQ|
#SH|# --------------------------------------------
#SB|# Application Settings
#RH|# --------------------------------------------
#HX|NODE_ENV=production
#HJ|PORT=3000
#XK|
#TK|# --------------------------------------------
#TY|# External Service Endpoints (Read-only)
#HN|# --------------------------------------------
#YV|# EXTERNAL_IP is used in docker-compose.yml via variable substitution
#TP|# Set this to your LLM/STT server IP address
#TS|EXTERNAL_IP=192.168.178.144
#RY|# --------------------------------------------
#ZH|```
#QT|
#WS|---
#XQ|
#PR|## Installation Steps
#QB|
#YB|### Phase 1: Preparation
#XS|
#HK|#### Step 1.1: Create Project Directory Structure
#YM|
#BV|```bash
#TQ|# SSH to Docker Host
#PP|ssh docker-a@192.168.178.142
#YH|
#XT|# Create project directory
#XK|mkdir -p /mnt/pve/data_2/filebrowser/open-notebook/{surreal_data,notebook_data,surreal,scripts}
#SN|cd /mnt/pve/data_2/filebrowser/open-notebook
#XN|
#SS|# Verify directory structure
#YN|ls -la
#RR|```
#NK|
#KB|**Acceptance Criteria:**
#BV|```bash
#RZ|# Expected output:
#BJ|# total 20
#TV|# drwxr-xr-x 5 user user 4096 Jun 14 10:00 .
#NS|# drwxr-xr-x 1 root root 4096 Jun 14 10:00 ..
#QR|# drwxr-xr-x 2 user user 4096 Jun 14 10:00 surreal_data
#BH|# drwxr-xr-x 2 user user 4096 Jun 14 10:00 notebook_data
#SR|# drwxr-xr-x 2 user user 4096 Jun 14 10:00 scripts
#TN|# drwxr-xr-x 2 user user 4096 Jun 14 10:00 surreal
#JX|```
#TH|
#HN|#### Step 1.2: Create Configuration Files
#MM|
#BV|```bash
#BH|# Create docker-compose.yml
#ZP|cat > docker-compose.yml << 'EOF'
#KN|# Paste content from Configuration Files section
#VR|EOF
#HY|
#RY|# Create .env file
#QR|cat > .env << 'EOF'
#KN|# Paste content from Configuration Files section
#VR|EOF
#TN|
#SH|# Set secure permissions on .env
#PR|chmod 600 .env
#WX|
#ZX|# Verify files created
#PP|ls -la docker-compose.yml .env
#VH|```
#JH|
#TM|#### Step 1.3: Initialize SurrealDB Namespace
#RS|
#BV|```bash
#ZH|# Start SurrealDB container first
#WT|docker compose up -d surrealdb
#RX|
#QP|# Wait for SurrealDB to be ready
#WB|sleep 5
#YP|
#BH|# Create namespace and database
#JP|docker compose exec surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} << 'EOSQL'
#YM|CREATE NAMESPACE open_notebook;
#HK|USE NS open_notebook;
#HT|CREATE DB open_notebook;
#ZJ|EOSQL
#SQ|
#TB|# Verify namespace created
#KH|docker compose exec surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} --output raw "INFO FOR NS;"
#SK|```
#QB|
#QP|**Acceptance Criteria**:
#RP|- Namespace `open_notebook` exists
#SR|- Database `open_notebook` created
#KP|
#JR|### Phase 1.5: Rollback Preparation
#MH|
#PT|#### Step 1.5.1: Pre-Installation Backup
#HN|
#BV|```bash
#JN|# Create backup of existing installation (if any)
#YS|if [ -d "/mnt/pve/data_2/filebrowser/open-notebook" ]; then
#QY|    echo "Found existing installation, creating backup..."
#BT|    BACKUP_DIR="/mnt/pve/data_2/filebrowser/open-notebook-pre-install-backup-$(date +%Y%m%d_%H%M%S)"
#WY|    cp -r /mnt/pve/data_2/filebrowser/open-notebook "$BACKUP_DIR"
#RX|    echo "Backup created: $BACKUP_DIR"
#HB|fi
#YV|```
#BR|
#HM|#### Step 1.5.2: Cleanup Script
#SV|
#BV|```bash
#PY|cat > scripts/cleanup.sh << 'EOF'
#JB|#!/bin/bash
#JB|
#WQ|echo "=== Open Notebook Cleanup Script ==="
#YN|echo "WARNING: This will stop and remove all containers"
#NB|echo ""
#MZ|
#WS|# Stop containers
#SX|docker compose down
#HM|
#ZP|# Remove volumes (optional - uncomment if you want to remove data)
#VM|# docker volume rm open-notebook_surreal_data
#VN|
#PV|# Remove images (optional)
#MK|# docker rmi surrealdb/surrealdb:v2.0.0
#BQ|# docker rmi anomalyco/opencode:latest
#XS|
#HP|echo "Cleanup complete"
#BM|echo "To reinstall, run: docker compose up -d"
#VR|EOF
#YS|
#KW|chmod +x scripts/cleanup.sh
#NN|```
#RS|
#TM|### Phase 2: Network Verification
#MJ|
#XN|#### Step 2.1: Test External Server Connectivity
#HT|
#BV|```bash
#VQ|# Create verification script
#YS|cat > scripts/verify-network.sh << 'EOF'
#JB|#!/bin/bash
#YY|
#TK|echo "=== Network Connectivity Verification ==="
#WY|echo "Date: $(date)"
#NB|echo ""
#XY|
#BJ|# Colors for output
#XM|RED='\033[0;31m'
#WX|GREEN='\033[0;32m'
#YH|NC='\033[0m' # No Color
#ZQ|
#YB|# Get EXTERNAL_IP from environment or .env
#QY|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#WW|
#YB|# Test LLM Server
#QY|echo "Testing LLM Server (${EXTERNAL_IP}:8082)..."
#WW|if curl -s --connect-timeout 5 http://${EXTERNAL_IP}:8082/v1/models > /dev/null 2>&1; then
#BY|    echo -e "${GREEN}✓ LLM Server is reachable${NC}"
#YV|    curl -s http://${EXTERNAL_IP}:8082/v1/models | jq '.'
#SQ|else
#QR|    echo -e "${RED}✗ LLM Server is NOT reachable${NC}"
#PJ|    exit 1
#HB|fi
#NB|echo ""
#PN|
#WS|# Test STT Server
#MH|echo "Testing STT Server (${EXTERNAL_IP}:9000)..."
#WH|if curl -s --connect-timeout 5 http://${EXTERNAL_IP}:9000/health > /dev/null 2>&1; then
#PT|    echo -e "${GREEN}✓ STT Server is reachable${NC}"
#SQ|else
#HQ|    echo -e "${YELLOW}⚠ STT Server health endpoint not found (may still work)${NC}"
#HB|fi
#NB|echo ""
#YY|
#NP|# Test Docker Host to itself
#WN|echo "Testing localhost connectivity..."
#JT|if curl -s --connect-timeout 2 http://localhost > /dev/null 2>&1; then
#KK|    echo -e "${GREEN}✓ Localhost is reachable${NC}"
#SQ|else
#TW|    echo -e "${RED}✗ Localhost connectivity issue${NC}"
#HB|fi
#NB|echo ""
#XM|
#XZ|echo "=== Network Verification Complete ==="
#VR|EOF
#TS|
#KS|chmod +x scripts/verify-network.sh
#HW|
#ZJ|# Run verification
#KN|./scripts/verify-network.sh
#XT|```
#HP|
#KB|**Acceptance Criteria:**
#TN|- LLM Server returns valid JSON with available models
#HT|- STT Server responds (health endpoint may vary by implementation)
#TH|- All tests complete within 10 seconds
#XX|
#MS|### Phase 3: Container Deployment
#HM|
#HN|#### Step 3.1: Pull Images
#QS|
#BV|```bash
#QZ|# Pull latest images (may take 5-10 minutes depending on bandwidth)
#MP|docker compose pull
#PW|
#XK|# Verify images downloaded
#VH|docker images | grep -E 'surrealdb|opencode'
#NP|```
#RJ|
#MS|**Expected Output:**
#SW|```
#PW|REPOSITORY          TAG       IMAGE ID       SIZE
#HX|surrealdb/surrealdb v2.0.0    abc123def456   500MB
#KK|anomalyco/opencode  latest    def456abc789   1.2GB
#BJ|```
#XY|
#YB|#### Step 3.2: Start Services
#BV|
#BV|```bash
#SQ|# Start all services
#WV|docker compose up -d
#SK|
#YQ|# Wait for services to initialize
#VB|sleep 10
#HS|
#QX|# Check container status
#XX|docker compose ps
#BZ|```
#HK|
#KB|**Acceptance Criteria:**
#NR|```
#KJ|NAME              STATUS              PORTS
#XR|surrealdb         Up (healthy)        8000/tcp
#HQ|open-notebook     Up (healthy)        0.0.0.0:3000->3000/tcp
#MW|```
#ZN|
#KP|#### Step 3.3: Verify Service Health
#QB|
#BV|```bash
#MQ|# Create health check script
#PV|cat > scripts/verify-services.sh << 'EOF'
#JB|#!/bin/bash
#YS|
#ZB|echo "=== Service Health Verification ==="
#WY|echo "Date: $(date)"
#NB|echo ""
#KX|
#JP|# Check SurrealDB
#ZB|echo "1. SurrealDB Health Check..."
#PY|if docker compose exec -T surrealdb curl -s http://localhost:8000/health > /dev/null 2>&1; then
#WB|    echo "   ✓ SurrealDB is healthy"
#SQ|else
#XX|    echo "   ✗ SurrealDB health check failed"
#JP|    docker compose logs surrealdb --tail 20
#PJ|    exit 1
#HB|fi
#BM|
#KN|# Check Open Notebook
#QP|echo "2. Open Notebook Health Check..."
#RW|if curl -s http://localhost:3000/api/health > /dev/null 2>&1; then
#XP|    echo "   ✓ Open Notebook is healthy"
#SQ|else
#WS|    echo "   ✗ Open Notebook health check failed"
#HM|    docker compose logs open-notebook --tail 20
#PJ|    exit 1
#HB|fi
#PW|
#PM|# Check external connectivity from Open Notebook container
#HS|echo "3. LLM Server Connectivity from Container..."
#BN|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#JZ|if docker compose exec -T open-notebook curl -s --connect-timeout 5 http://${EXTERNAL_IP}:8082/v1/models > /dev/null 2>&1; then
#JZ|    echo "   ✓ Can reach LLM Server from container"
#SQ|else
#XH|    echo "   ✗ Cannot reach LLM Server from container"
#PJ|    exit 1
#HB|fi
#KZ|
#NB|echo ""
#ZS|echo "=== All Health Checks Passed ==="
#VR|EOF
#JV|
#VN|chmod +x scripts/verify-services.sh
#RY|./scripts/verify-services.sh
#RB|```
#PM|
#TV|### Phase 4: Initial Configuration
#PZ|
#QX|#### Step 4.1: Access Open Notebook UI
#SK|
#BV|```bash
#TJ|# Open browser to:
#PN|# http://192.168.178.142:3000
#YJ|```
#MR|
#HK|#### Step 4.2: Configure LLM Provider
#JV|
#XT|Navigate to: **Settings → AI Providers → Add Custom Provider**
#PS|
#VT|**Configuration:**
#YN|```
#HT|Provider Name: Local Llama-Swap
#TT|API Endpoint: http://${EXTERNAL_IP}:8082/v1
#NH|API Key: (leave empty if not required, or enter your key)
#KM|Model: gemma2:31b
#VH|Format: OpenAI-compatible
#TP|```
#HQ|
#SJ|#### Step 4.3: Configure STT Provider
#PM|
#NN|Navigate to: **Settings → Speech-to-Text → Add Provider**
#VN|
#VT|**Configuration:**
#KR|```
#JR|Provider Name: WhisperX Local
#BX|API Endpoint: http://${EXTERNAL_IP}:9000
#JB|API Key: (leave empty if not required)
#VZ|Language: auto-detect
#NP|Model: large-v3 (or your preferred model)
#HW|```
#ZT|
#MJ|#### Step 4.4: Model Discovery
#RR|
#MT|Navigate to: **Settings → Models → Refresh/Discover**
#SN|
#HQ|**Expected Result:**
#KP|- gemma2:31b appears in available models list
#QS|- Model shows as "ready" or "available"
#MV|
#KN|---
#XJ|
#WH|## Network Verification
#JX|
#WZ|### Complete Network Test Suite
#XN|
#BV|```bash
#QR|# Comprehensive network verification
#XB|cat > scripts/full-network-test.sh << 'EOF'
#JB|#!/bin/bash
#XS|
#RH|echo "=========================================="
#KM|echo "  Full Network Connectivity Test Suite  "
#RH|echo "=========================================="
#NB|echo ""
#PX|
#HS|# Test 1: Docker Host network configuration
#TZ|echo "TEST 1: Docker Host Network Configuration"
#JS|echo "----------------------------------------"
#VB|ip addr show | grep -E 'inet |eth0|ens'
#NB|echo ""
#ZH|
#KK|# Test 2: DNS resolution (if applicable)
#NQ|echo "TEST 2: DNS Resolution"
#YB|echo "----------------------"
#VM|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#NB|nslookup ${EXTERNAL_IP} || echo "nslookup not available, skipping"
#NB|echo ""
#XW|
#PZ|# Test 3: Port availability
#NK|echo "TEST 3: Port Availability Check"
#YJ|echo "--------------------------------"
#BV|for port in 8082 9000; do
#QP|    if nc -zv ${EXTERNAL_IP} $port 2>&1 | grep -q succeeded; then
#XK|        echo "✓ Port $port on ${EXTERNAL_IP} is open"
#ZR|    else
#XK|        echo "✗ Port $port on ${EXTERNAL_IP} is NOT accessible"
#MT|    fi
#PX|done
#NB|echo ""
#NY|
#RQ|# Test 4: HTTP/HTTPS connectivity
#SJ|echo "TEST 4: HTTP Service Connectivity"
#NT|echo "----------------------------------"
#ZR|echo "LLM Server (/v1/models):"
#TP|curl -s -o /dev/null -w "  Status: %{http_code}\n" http://${EXTERNAL_IP}:8082/v1/models
#YN|
#RQ|echo "STT Server (health check):"
#VX|curl -s -o /dev/null -w "  Status: %{http_code}\n" http://${EXTERNAL_IP}:9000/health 2>/dev/null || echo "  Health endpoint not available"
#NB|echo ""
#WM|
#MQ|# Test 5: Container network access
#NP|echo "TEST 5: Container Network Access"
#TR|echo "---------------------------------"
#YR|echo "From surrealdb container:"
#WK|docker compose exec -T surrealdb ping -c 2 ${EXTERNAL_IP} 2>&1 | grep -E 'bytes from|ping'
#RW|
#KQ|echo "From open-notebook container:"
#TV|docker compose exec -T open-notebook ping -c 2 ${EXTERNAL_IP} 2>&1 | grep -E 'bytes from|ping'
#NB|echo ""
#XY|
#SV|# Test 6: Performance test
#YX|echo "TEST 6: Response Time Test"
#NX|echo "--------------------------"
#HV|echo "LLM Server response time:"
#WJ|start_time=$(date +%s%N)
#SY|curl -s http://${EXTERNAL_IP}:8082/v1/models > /dev/null
#XN|end_time=$(date +%s%N)
#XZ|response_time=$((($end_time - $start_time) / 1000000))
#WX|echo "  Response time: ${response_time}ms"
#NB|echo ""
#YX|
#RH|echo "=========================================="
#XV|echo "  Test Suite Complete                    "
#RH|echo "=========================================="
#VR|EOF
#WS|
#VS|chmod +x scripts/full-network-test.sh
#JP|./scripts/full-network-test.sh
#ZK|```
#PR|
#HR|**Expected Results:**
#NK|- All ports (8082, 9000) show as accessible
#YV|- HTTP status codes: 200 for both services
#RM|- Container pings succeed (< 1ms latency expected on LAN)
#JT|- Response time < 100ms for LLM models endpoint
#BJ|
#MY|---
#MQ|
#WM|## Model Discovery & Registration
#NQ|
#RZ|### Step 1: Query Available Models
#ZK|
#BV|```bash
#RX|# Query llama-swap for available models
#WH|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#WN|curl -s http://${EXTERNAL_IP}:8082/v1/models | jq '.'
#XN|```
#XN|
#WQ|**Expected Response:**
#YP|```json
#WH|{
#JZ|  "object": "list",
#RM|  "data": [
#HP|    {
#JT|      "id": "gemma2:31b",
#VJ|      "object": "model",
#HN|      "created": 1234567890,
#ZM|      "owned_by": "llama-swap",
#JM|      "permission": []
#QH|    }
#MR|  ]
#RJ|}
#TH|```
#KW|
#TJ|### Step 2: Test Model Inference
#XB|
#BV|```bash
#VQ|# Test basic inference
#NB|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#NB|curl -s http://${EXTERNAL_IP}:8082/v1/chat/completions \
#VQ|  -H "Content-Type: application/json" \
#BW|  -d '{
#PP|    "model": "gemma2:31b",
#VQ|    "messages": [
#ZT|      {
#XJ|        "role": "user",
#MN|        "content": "Hello, this is a test."
#RR|      }
#XW|    ],
#MN|    "max_tokens": 50
#KN|  }' | jq '.choices[0].message.content'
#BV|```
#WZ|
#YN|**Expected:** Returns a text response within 5-10 seconds
#QB|
#BX|### Step 3: Register Model in Open Notebook
#SS|
#WB|After accessing the UI:
#HY|
#YV|1. Navigate to **Settings → Models**
#MZ|2. Click **Add Model** or **Discover Models**
#KH|3. Enter model details:
#ST|   - **Model ID:** `gemma2:31b`
#JB|   - **Model Name:** Gemma2 31B
#XR|   - **Provider:** Local Llama-Swap
#NT|   - **Context Window:** 8192 (or as appropriate)
#JP|   - **Temperature:** 0.7 (default)
#YQ|
#PH|---
#VT|
#MX|## Testing & Verification
#YM|
#JJ|### Functional Test Suite
#NN|
#BV|```bash
#YS|# Create comprehensive test script
#NW|cat > scripts/functional-tests.sh << 'EOF'
#JB|#!/bin/bash
#RN|
#RH|echo "=========================================="
#TS|echo "  Open Notebook Functional Tests        "
#RH|echo "=========================================="
#NB|echo ""
#ZZ|
#BP|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#BM|TARGET_HOST=${EXTERNAL_IP}
#JM|
#MX|# Test 1: API Health Endpoint
#XR|echo "TEST 1: API Health Check"
#MT|echo "------------------------"
#XZ|response=$(curl -s "http://${TARGET_HOST}:3000/api/health")
#HY|if echo "$response" | grep -q "healthy\|ok\|up"; then
#VT|    echo "✓ API is healthy"
#RZ|    echo "$response" | jq '.'
#SQ|else
#YP|    echo "✗ API health check failed"
#PZ|    echo "$response"
#HB|fi
#NB|echo ""
#VJ|
#RP|# Test 2: GraphQL Endpoint (if enabled)
#RV|echo "TEST 2: GraphQL Endpoint"
#MT|echo "------------------------"
#PH|if curl -s -X POST "http://${TARGET_HOST}:3000/graphql" \
#VQ|  -H "Content-Type: application/json" \
#XY|  -d '{"query": "{ __schema { types { name } } }"}' > /dev/null 2>&1; then
#RT|    echo "✓ GraphQL endpoint is accessible"
#SQ|else
#XZ|    echo "✗ GraphQL endpoint not accessible (may be disabled)"
#HB|fi
#NB|echo ""
#SP|
#ZZ|# Test 3: LLM Integration Test
#ZV|echo "TEST 3: LLM Integration"
#QZ|echo "-----------------------"
#BX|# This would require authentication token from actual UI session
#HQ|echo "ℹ Manual test required: Use UI to send a message"
#JT|echo "   Expected: Response from gemma2:31b within 10-30 seconds"
#NB|echo ""
#KV|
#QH|# Test 4: STT Integration Test
#TM|echo "TEST 4: STT Integration"
#QZ|echo "-----------------------"
#YW|echo "ℹ Manual test required: Upload audio file via UI"
#JJ|echo "   Expected: Transcription within 5-15 seconds"
#NB|echo ""
#TM|
#PY|# Test 5: Database Connectivity
#JP|echo "TEST 5: Database Connectivity"
#NQ|echo "-----------------------------"
#YB|if docker compose exec -T surrealdb surreal sql --endpoint http://localhost:8000 --namespace open_notebook --user root --pass "${SURREALDB_PASSWORD}" --output raw "INFO FOR DB;" > /dev/null 2>&1; then
#XQ|    echo "✓ SurrealDB is accessible"
#SQ|else
#RT|    echo "✗ SurrealDB connectivity issue"
#HB|fi
#NB|echo ""
#SX|
#QM|# Test 6: Container Resource Usage
#MH|echo "TEST 6: Resource Usage"
#YB|echo "----------------------"
#ZJ|docker compose ps --format table
#NB|echo ""
#HW|docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
#NB|echo ""
#WR|
#RH|echo "=========================================="
#YB|echo "  Functional Tests Complete             "
#RH|echo "=========================================="
#NB|echo ""
#HB|echo "Note: Some tests require manual verification via UI"
#PH|echo "UI Access: http://${TARGET_HOST}:3000"
#VR|EOF
#TK|
#RY|chmod +x scripts/functional-tests.sh
#MX|./scripts/functional-tests.sh
#RJ|```
#HQ|
#JJ|### Performance Benchmarks
#SV|
#SS|**Expected Performance:**
#ZP|- **LLM Response Time:** 5-30 seconds for typical queries (depends on model size and server load)
#PT|- **STT Transcription:** 5-15 seconds for 1-minute audio clip
#TT|- **Database Queries:** < 100ms for typical operations
#QP|- **UI Load Time:** < 2 seconds on local network
#RK|
#NY|---
#RK|
#YQ|## Security Hardening
#BR|
#VR|### Phase 1: Initial Setup (Current)
#SQ|
#SS|**Current Security Posture:**
#KH|- Password-based authentication for SurrealDB
#WW|- No encryption at rest (planned migration)
#VV|- No TLS/SSL (internal network only)
#PN|- API keys stored in .env file
#NZ|
#VJ|### Phase 2: Encryption Key Migration
#QH|
#BV|```bash
#WT|# Generate secure encryption key
#SN|openssl rand -hex 32
#VS|
#VR|# Output: 64-character hex string (256 bits)
#PH|# Example: a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456
#WQ|```
#BK|
#QJ|**Update .env File:**
#RP|
#BV|```bash
#JW|# Edit .env file
#SH|nano .env
#BX|
#QY|# Update ENCRYPTION_KEY line:
#XP|ENCRYPTION_KEY=a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456
#JW|
#QV|# Save and exit (Ctrl+X, Y, Enter)
#NK|```
#TK|
#WW|**Apply Changes:**
#TW|
#BV|```bash
#XS|# Restart containers to apply new encryption key
#SX|docker compose down
#WV|docker compose up -d
#PV|
#YY|# Verify services are healthy
#XX|docker compose ps
#PV|```
#KT|
#MR|### Phase 3: Password Hardening
#RY|
#BV|```bash
#YW|# Generate strong SurrealDB password
#MV|openssl rand -base64 32
#MB|
#NR|# Update .env file
#SH|nano .env
#NJ|
#BK|# Update SURREALDB_PASSWORD line with new password
#KZ|SURREAL_PASSWORD=NewSecurePassword123!
#KY|
#QT|# Restart services
#SX|docker compose down
#WV|docker compose up -d
#YW|```
#WW|
#HV|### Phase 4: Additional Security Measures
#JB|
#ZQ|**Recommended Future Enhancements:**
#NQ|
#BH|1. **Enable TLS/SSL** (when firewall is added):
#XH|   - Add reverse proxy (nginx/Traefik) with Let's Encrypt certificates
#VN|   - Update container ports to use HTTPS
#WN|
#RZ|2. **Implement Network Segmentation**:
#XS|   - Create Docker bridge network instead of host mode
#BY|   - Add firewall rules between network segments
#VY|
#JQ|3. **Regular Security Audits**:
#YY|   - Schedule monthly vulnerability scans
#SQ|   - Monitor container logs for suspicious activity
#KN|
#ZB|4. **Backup Encryption**:
#WX|   - Encrypt backup files with GPG
#PN|   - Store encryption keys separately from backups
#XP|
#HQ|---
#PS|
#VS|## Backup & Maintenance
#PW|
#MZ|### Backup Strategy
#NQ|
#PB|**Proxmox Backup Server Approach (Recommended):**
#NK|Since this installation runs on a Proxmox host with access to Proxmox Backup Server (PBS), the recommended backup strategy is:
#BS|
#PT|1. **VM/Container Backups**: Use Proxmox Backup Server for full system backups
#HM|- Schedule daily incremental backups
#PT|- Weekly full backups
#HM|- Retention: 7 days incremental, 4 weeks full
#PT|
#BS|2. **Volume Backups**: PBS automatically backs up Docker volumes
#BS|- surreal_data volume will be included
#PT|- notebook_data volume will be included
#BS|
#PT|3. **Configuration Backups**: Git version control
#HM|- Commit docker-compose.yml and .env to private repo
#PT|- Store secrets in encrypted format or use Proxmox vault
#BS|
#PT|**Manual Backups (Optional):**
#BS|If you need additional manual backups outside of PBS:
#PT|
#BS|```bash
#BS|# Create manual backup (optional)
#BS|cat > scripts/manual-backup.sh << 'EOF'
#BS|#!/bin/bash
#BS|DATE=$(date +%Y%m%d_%H%M%S)
#BS|BACKUP_DIR="/mnt/pve/data_2/filebrowser/open-notebook/backups"
#BS|
#BS|mkdir -p "$BACKUP_DIR"
#BS|
#BS|# Backup SurrealDB data
#BS|echo "Backing up SurrealDB..."
#BS|docker exec surrealdb surreal dump > "${BACKUP_DIR}/surrealdb_${DATE}.sql"
#BS|
#BS|# Backup configuration
#BS|echo "Backing up configuration..."
#BS|tar -czf "${BACKUP_DIR}/config_${DATE}.tar.gz" .env docker-compose.yml
#BS|
#BS|echo "Backup complete: ${BACKUP_DIR}"
#BS|EOF
#BS|
#BS|chmod +x scripts/manual-backup.sh
#BS|```
#BS|
#PT|**Note**: The backup.sh and restore.sh scripts have been removed as Proxmox Backup Server provides superior backup capabilities with deduplication, compression, and integrated restore workflows.
#BS|
#HB|---
#JQ|
#NH|### Maintenance Procedures
#RT|
#VT|#### Daily Monitoring
#YY|
#BV|```bash
#HK|# Check container health
#WZ|docker compose ps
#KS|
#SZ|# View recent logs
#XP|docker compose logs --tail 50
#MX|
#RH|# Check resource usage
#BR|docker stats --no-stream
#HT|```
#HP|
#MM|#### Weekly Maintenance
#RS|
#BV|```bash
#BH|# Create weekly maintenance script
#ZP|cat > scripts/weekly-maintenance.sh << 'EOF'
#KN|#!/bin/bash
#VR|
#HY|echo "=========================================="
#RY|echo "  Weekly Maintenance Tasks              "
#HY|echo "=========================================="
#TP|
#PK|# 1. Check disk space
#NB|echo ""
#HZ|echo "1. Disk space check..."
#VT|df -h / | tail -1
#SR|
#WT|# 2. Check for container restarts
#NB|echo ""
#YY|echo "2. Container status..."
#NM|docker compose ps --format json | jq -r '.[] | "\(.Name): \(.Status)"'
#VS|
#YM|# 3. Review error logs
#NB|echo ""
#BT|echo "3. Recent errors (last 24 hours)..."
#VQ|docker compose logs --since 24h --tail 100 | grep -i error | tail -20
#WP|
#NB|echo ""
#RH|echo "=========================================="
#SM|echo "  Maintenance Complete                  "
#RH|echo "=========================================="
#VR|EOF
#VS|
#YX|chmod +x scripts/weekly-maintenance.sh
#JH|```
#MX|
#QW|#### Monthly Updates
#QV|
#BV|```bash
#RZ|# Update procedure
#XM|# 1. Check for new images
#MP|docker compose pull
#MT|
#RY|# 2. Review changelog
#JM|# Check anomalyco/opencode releases
#JQ|
#MJ|# 3. Schedule maintenance window
#SX|docker compose down
#QY|
#KR|# 4. Start with new images
#WV|docker compose up -d
#WB|
#PN|# 5. Verify health
#RY|./scripts/verify-services.sh
#WW|```
#TH|
#JH|---
#BT|
#ZH|## Troubleshooting Guide
#WM|
#XK|### Common Issues & Solutions
#RJ|
#QP|#### Issue 1: Containers Won't Start
#WR|
#TJ|**Symptoms:**
#HV|```
#SB|Error: Cannot start service surrealdb: driver failed
#VV|```
#BV|
#HT|**Diagnosis:**
#BV|```bash
#KH|# Check Docker daemon status
#ZW|systemctl status docker
#MS|
#JN|# Check port conflicts
#SJ|netstat -tlnp | grep -E '8000|3000'
#MW|
#JZ|# Check system logs
#TJ|journalctl -xe | grep docker
#HP|```
#JS|
#BZ|**Solutions:**
#HT|1. Ensure Docker is running: `systemctl start docker`
#BM|2. Free up ports if in use
#QQ|3. Check available memory: `free -h`
#YP|
#JX|---
#JQ|
#BS|#### Issue 2: Cannot Connect to LLM Server
#WS|
#TJ|**Symptoms:**
#JX|- LLM requests timeout
#YT|- "Network error" in UI
#VB|
#HT|**Diagnosis:**
#BV|```bash
#PM|# Test from Docker Host
#XP|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#SV|curl -v http://${EXTERNAL_IP}:8082/v1/models
#SV|
#KV|# Test from container
#RT|docker compose exec open-notebook curl -v http://${EXTERNAL_IP}:8082/v1/models
#MV|
#XV|# Check container network
#MV|docker compose exec open-notebook ip addr show
#QS|```
#JM|
#BZ|**Solutions:**
#XT|1. Verify host network mode in docker-compose.yml
#HT|2. Check firewall on LLM Server: `sudo ufw status`
#JS|3. Verify llama-swap is running: `systemctl status llama-swap`
#KB|
#ZM|---
#VW|
#WS|#### Issue 3: STT Not Working
#XH|
#TJ|**Symptoms:**
#BT|- Audio upload fails
#TP|- Transcription returns error
#ZP|
#HT|**Diagnosis:**
#BV|```bash
#HY|# Test STT endpoint directly
#ZH|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#ST|curl -v http://${EXTERNAL_IP}:9000/health
#ST|
#TS|# Check WhisperX logs (if accessible)
#KX|ssh docker-a@192.168.178.144
#QX|sudo journalctl -u whisperx -n 50
#XW|
#XW|# Verify NVIDIA GPU access
#KX|ssh docker-a@192.168.178.144
#BY|nvidia-smi
#VM|```
#HJ|
#BZ|**Solutions:**
#PK|1. Ensure WhisperX API endpoint is correct
#TY|2. Check API key configuration
#PW|3. Verify GPU is accessible and not overloaded
#WT|
#YT|---
#RB|
#KQ|#### Issue 4: Slow Performance
#XT|
#TJ|**Symptoms:**
#JN|- LLM responses take > 60 seconds
#HH|- UI is sluggish
#YV|
#HT|**Diagnosis:**
#BV|```bash
#XZ|# Check container resource usage
#MT|docker stats
#YR|
#TK|# Check LLM Server load
#KX|ssh docker-a@192.168.178.144
#KR|htop
#WX|
#MS|# Check network latency
#SV|ping -c 5 ${EXTERNAL_IP}
#BZ|```
#PN|
#BZ|**Solutions:**
#BY|1. Reduce concurrent requests to LLM Server
#RH|2. Check if other processes are using GPU
#KH|3. Consider upgrading hardware or using smaller model
#YR|
#NM|---
#TN|
#TZ|#### Issue 5: Database Connection Errors
#JH|
#TJ|**Symptoms:**
#XQ|- "Cannot connect to database" error
#KV|- Data not persisting
#BZ|
#HT|**Diagnosis:**
#BV|```bash
#XR|# Check SurrealDB logs
#YP|docker compose logs surrealdb --tail 50
#RR|
#NX|# Test SurrealDB connection
#NB|docker compose exec surrealdb curl -v http://localhost:8000/health
#RY|
#WR|# Check data directory permissions
#YH|ls -la surreal_data/
#JH|```
#HT|
#BZ|**Solutions:**
#PT|1. Ensure data directory has correct permissions: `chmod 755 surreal_data`
#SZ|2. Check SurrealDB password in .env matches command
#PZ|3. Restart SurrealDB: `docker compose restart surrealdb`
#NH|
#NW|---
#VP|
#VX|#### Issue 6: Host Network Mode Issues
#PP|
#TJ|**Symptoms:**
#SW|- Port conflicts with other services
#RQ|- Cannot bind to specific ports
#TM|
#BZ|**Solutions:**
#RW|1. Check which ports are in use: `netstat -tlnp`
#HK|2. Consider switching to bridge network mode if conflicts persist
#SV|3. Update docker-compose.yml ports mapping
#YX|
#TH|---
#SK|
#WN|### Log Analysis Commands
#BY|
#BV|```bash
#QX|# Real-time log monitoring
#WV|docker compose logs -f
#HB|
#PM|# Search for errors
#KW|docker compose logs | grep -i error
#TT|
#TY|# View specific service logs
#ST|docker compose logs open-notebook --tail 100
#PT|
#QP|# Export logs for analysis
#HH|docker compose logs > /tmp/opencode-logs-$(date +%Y%m%d).txt
#MN|```
#QW|
#WQ|---
#YP|
#TB|## Acceptance Criteria
#RQ|
#SB|### ✅ Installation Verification
#QB|
#BV|```bash
#TP|# Criterion 1: All containers running
#TJ|docker compose ps | grep -E "Up|healthy"
#QV|# Expected: Both surrealdb and open-notebook show "Up" status
#WQ|
#XQ|# Criterion 2: Services healthy
#RY|./scripts/verify-services.sh
#TQ|# Expected: All health checks pass
#RP|
#MT|# Criterion 3: External connectivity
#KN|./scripts/verify-network.sh
#RS|# Expected: LLM and STT servers reachable
#WR|```
#TJ|
#YQ|# Criterion 4: External services health
#PX|./scripts/verify-external-services.sh
#QY|# Expected: All external services respond
#KN|./scripts/verify-network.sh
#RS|# Expected: LLM and STT servers reachable
#VZ|```
#TW|
#VS|### ✅ Functional Verification
#QY|
#BV|```bash
#VP|# Criterion 4: UI accessible
#PY|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#JS|curl -s http://${EXTERNAL_IP}:3000 | grep -i "open notebook"
#JS|# Expected: HTML page returned
#XZ|
#BM|# Criterion 5: API responsive
#KB|curl -s http://${EXTERNAL_IP}:3000/api/health
#WJ|# Expected: {"status":"healthy"} or similar
#YR|
#JP|# Criterion 6: LLM integration
#ZM|# Manual: Send message via UI, expect response within 30 seconds
#QB|# Expected: Valid response from gemma2:31b
#XP|
#SW|# Criterion 7: STT integration
#BB|# Manual: Upload audio file, expect transcription
#ZJ|# Expected: Transcription within 15 seconds for 1-min audio
#SN|```
#NX|
#ZT|# Criterion 8: Database namespace initialized
#QS|docker compose exec -T surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} --output raw "INFO FOR NS;" | grep -q "open_notebook"
#RQ|if [ $? -eq 0 ]; then
#XM|    echo "✓ Namespace open_notebook exists"
#SQ|else
#VB|    echo "✗ Namespace open_notebook NOT found"
#PJ|    exit 1
#HB|fi
#BB|# Manual: Upload audio file, expect transcription
#ZJ|# Expected: Transcription within 15 seconds for 1-min audio
#TV|```
#QX|
#YP|### ✅ Security Verification
#WY|
#BV|```bash
#HW|# Criterion 8: .env file permissions
#ZS|ls -la .env
#BV|# Expected: -rw------- (600)
#BP|
#XN|# Criterion 9: Encryption key set
#QM|grep "ENCRYPTION_KEY=" .env | grep -v "your-32-byte"
#SW|# Expected: Actual key value present
#MM|
#VV|# Criterion 10: Strong passwords
#QS|grep "SURREALDB_PASSWORD" .env | grep -v "ChangeMe"
#VY|# Expected: Strong password present
#HN|```
#BN|
#TN|# Criterion 11: Persistence configured
#HW|docker compose exec -T surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} --output raw "SELECT * FROM open_notebook.info;" > /dev/null 2>&1
#RQ|if [ $? -eq 0 ]; then
#MR|    echo "✓ Database persistence verified"
#SQ|else
#WR|    echo "✗ Database persistence NOT working"
#PJ|    exit 1
#HB|fi
#QS|grep "SURREALDB_PASSWORD" .env | grep -v "ChangeMe"
#VY|# Expected: Strong password present
#VM|```
#MW|
#RV|### ✅ Performance Verification
#TT|
#QW|| Metric | Target | Measurement Method |
#YK||--------|--------|-------------------|
#MZ|| UI Load Time | < 2s | Browser performance tab |
#SV|| LLM Response | < 30s | Manual test query |
#KB|| STT Transcription | < 15s/min | Upload 1-min audio |
#HM|| DB Query Time | < 100ms | API response time |
#MS|| Container Startup | < 60s | `docker compose up` |
#JP|
#VM|---
#VZ|
#SJ|## Automated Test Suite
#KH|
#BV|```bash
#VZ|# Run complete automated test suite
#KP|cat > scripts/run-all-tests.sh << 'EOF'
#JB|#!/bin/bash
#KK|
#BW|set -e
#NY|
#PT|echo "========================================"
#NJ|echo "  Open Notebook Automated Test Suite  
#PT|echo "========================================"
#NB|echo ""
#KZ|
#WR|PASS=0
#NZ|FAIL=0
#JV|
#HV|# Test 1: Container health
#PH|echo "[1/8] Container Health Check"
#PP|if docker compose ps | grep -q "Up"; then
#SN|    echo "  ✓ All containers running"
#SP|    ((PASS++))
#SQ|else
#ZW|    echo "  ✗ Some containers not running"
#JJ|    ((FAIL++))
#HB|fi
#VY|
#MJ|# Test 2: Service health
#JT|echo "[2/8] Service Health Check"
#ST|if ./scripts/verify-services.sh > /dev/null 2>&1; then
#HS|    echo "  ✓ All services healthy"
#SP|    ((PASS++))
#SQ|else
#JX|    echo "  ✗ Service health check failed"
#JJ|    ((FAIL++))
#HB|fi
#XN|
#HT|# Test 3: External connectivity
#MY|echo "[3/8] External Connectivity"
#MS|export EXTERNAL_IP=${EXTERNAL_IP:-192.168.178.144}
#RB|if ./scripts/verify-network.sh > /dev/null 2>&1; then
#RB|    echo "  ✓ External services reachable"
#SP|    ((PASS++))
#SQ|else
#RB|    echo "  ✗ External connectivity failed"
#JJ|    ((FAIL++))
#HB|fi
#JR|
#QW|# Test 4: External service health
#TM|echo "[4/8] External Service Health"
#YM|if ./scripts/verify-external-services.sh > /dev/null 2>&1; then
#HK|    echo "  ✓ All external services healthy"
#SP|    ((PASS++))
#SQ|else
#TN|    echo "  ✗ External service health check failed"
#JJ|    ((FAIL++))
#HB|fi
#WW|
#ZR|# Test 5: UI accessibility
#YV|echo "[5/8] UI Accessibility"
#BW|if curl -s http://${EXTERNAL_IP}:3000 | grep -qi "open notebook"; then
#ZX|    echo "  ✓ UI is accessible"
#SP|    ((PASS++))
#SQ|else
#XR|    echo "  ✗ UI not accessible"
#JJ|    ((FAIL++))
#HB|fi
#NX|
#VR|# Test 6: API health
#WZ|echo "[6/8] API Health Check"
#BT|API_RESPONSE=$(curl -s http://${EXTERNAL_IP}:3000/api/health)
#BP|if echo "$API_RESPONSE" | grep -qi "healthy\|ok\|up"; then
#ZB|    echo "  ✓ API is healthy"
#SP|    ((PASS++))
#SQ|else
#YP|    echo "  ✗ API health check failed"
#JJ|    ((FAIL++))
#HB|fi
#HT|
#PK|# Test 7: Database namespace
#JS|echo "[7/8] Database Namespace"
#SW|if docker compose exec -T surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} --output raw "INFO FOR NS;" 2>/dev/null | grep -q "open_notebook"; then
#HM|    echo "  ✓ Namespace open_notebook exists"
#SP|    ((PASS++))
#SQ|else
#WY|    echo "  ✗ Namespace NOT found"
#JJ|    ((FAIL++))
#HB|fi
#NH|
#NK|# Test 8: Security configuration
#MP|echo "[8/8] Security Configuration"
#KS|SECURITY_OK=true
#WB|if ! grep -q "ENCRYPTION_KEY=" .env | grep -v "your-32-byte"; then
#QY|    echo "  ✗ Encryption key not set"
#BH|    SECURITY_OK=false
#HB|fi
#ZJ|if ! grep -q "SURREALDB_PASSWORD=" .env | grep -v "ChangeMe"; then
#XR|    echo "  ✗ Weak password detected"
#BH|    SECURITY_OK=false
#HB|fi
#WQ|if [ "$SECURITY_OK" = true ]; then
#KY|    echo "  ✓ Security configuration valid"
#SP|    ((PASS++))
#SQ|else
#JJ|    ((FAIL++))
#HB|fi
#TH|
#NB|echo ""
#PT|echo "========================================"
#KY|echo "  Test Results: $PASS passed, $FAIL failed"
#PT|echo "========================================"
#BW|
#PT|if [ $FAIL -gt 0 ]; then
#XV|    echo "FAILED: Some tests did not pass"
#PJ|    exit 1
#SQ|else
#HS|    echo "SUCCESS: All tests passed!"
#BH|    exit 0
#HB|fi
#VR|EOF
#BZ|
#JW|chmod +x scripts/run-all-tests.sh
#QP|
#QS|# Execute the test suite
#SZ|./scripts/run-all-tests.sh
#RX|```
#XK|
#PS|---
#VP|
#PY|## Rollback Procedures
#JJ|
#SQ|### Full Rollback (Complete Reinstall)
#MM|
#BV|```bash
#JK|# 1. Stop all services
#SX|docker compose down
#PV|
#WH|# 2. Backup current state
#RM|Proxmox Backup Server handles this automatically
#RW|
#ZT|# 3. Run cleanup script
#HK|./scripts/cleanup.sh
#WS|
#HT|# 4. Remove remaining data (if needed)
#QJ|rm -rf /mnt/pve/data_2/filebrowser/open-notebook/surreal_data/*
#KQ|
#JY|# 5. Restore from Proxmox Backup (if needed)
#XZ|# Use Proxmox UI to restore from backup
#WN|
#JY|# 6. Reinstall
#BV|# Re-run installation steps from Phase 1
#VX|```
#SV|
#BZ|### Partial Rollback (Service Restart)
#HX|
#BV|```bash
#ZW|# Restart specific service
#WB|docker compose restart surrealdb
#ZY|# or
#QY|docker compose restart open-notebook
#RP|
#BN|# Reset to last known good state
#SX|docker compose down
#WV|docker compose up -d
#VT|```
#RR|
#HT|## Post-Installation Checklist
#BY|
#HM|- [ ] All containers running and healthy
#ZV|- [ ] UI accessible at http://192.168.178.142:3000
#JK|- [ ] LLM provider configured and tested
#VW|- [ ] STT provider configured and tested
#XK|- [ ] Model discovery completed
#YH|- [ ] Namespace open_notebook initialized
#JT|- [ ] First note created successfully
#KW|- [ ] Proxmox Backup Server backup configured
#QT|- [ ] Encryption key generated and applied
#NW|- [ ] Strong passwords set for all services
#TH|- [ ] External service health checks passing
#WR|- [ ] Automated test suite passing
#JB|- [ ] Rollback procedure tested
#SH|- [ ] Documentation updated with actual configuration
#PJ|
#WH|---
#WT|
#VB|## Appendix A: Quick Reference Commands
#PP|
#BV|```bash
#NZ|# Start services
#WV|docker compose up -d
#NV|
#ZP|# Stop services
#SX|docker compose down
#SB|
#QT|# Restart services
#PZ|docker compose restart
#RV|
#PW|# View logs
#WV|docker compose logs -f
#BJ|
#BW|# Check status
#XX|docker compose ps
#PP|
#BM|# Run verification
#RY|./scripts/verify-services.sh
#VR|
#MS|# Run all automated tests
#SZ|./scripts/run-all-tests.sh
#VX|
#RS|# Check external services
#PX|./scripts/verify-external-services.sh
#WZ|
#HP|# Rollback (cleanup)
#HK|./scripts/cleanup.sh
#QV|
#BM|# Initialize namespace (if needed)
#TY|docker compose exec surrealdb surreal sql --endpoint http://localhost:8000 --user root --pass ${SURREALDB_PASSWORD} "CREATE NAMESPACE open_notebook; USE NS open_notebook; CREATE DB open_notebook;"
#RY|./scripts/verify-services.sh
#QQ|
#XJ|# View resource usage
#MT|docker stats
#JH|
#KV|# Update to latest images
#PN|docker compose pull && docker compose up -d
#KK|```
#SK|
#RV|---
#NX|
#ST|## Appendix B: File Locations
#QQ|
#BJ|```
#VP|/mnt/pve/data_2/filebrowser/open-notebook/
#KY|├── docker-compose.yml      # Main configuration
#YK|├── .env                    # Environment variables (SENSITIVE)
#KN|├── surreal_data/           # SurrealDB persistence (mounted volume)
#MB|│   └── data.db             # Database file
#MS|├── notebook_data/          # Application data (mounted volume)
#PW|├── surreal/                # Legacy directory (can be removed)
#NY|├── scripts/
#PW|│   ├── verify-network.sh   # Network testing
#NY|│   ├── verify-services.sh  # Health checks
#RW|│   ├── functional-tests.sh # Functional testing
#KY|│   ├── cleanup.sh          # Cleanup/rollback
#SP|│   └── weekly-maintenance.sh # Weekly tasks
#SV|└── backups/                # Optional manual backups
#YJ|    └── *.tar.gz            # Manual backup files (if used)
#SZ|```
#MJ|
#TX|---
#RR|
#RR|## Appendix C: Network Diagram
#WY|
#BH|```
#NV|Internet
#SH|    │
#SW|    ▼
#SQ|┌─────────────────┐
#BN|│  Firewall       │ (None - all traffic allowed)
#YR|└────────┬────────┘
#BY|         │
#PW|         ▼
#SR|┌─────────────────────────────────┐
#HR|│   Router (192.168.178.1)        │
#KY|└────────┬────────────────────────┘
#NY|         │
#RR|    ┌────┴────┬──────────────┬──────────────┐
#TP|    │         │              │              │
#TR|    ▼         ▼              ▼              ▼
#SX|┌────────┐ ┌────────┐   ┌────────┐   ┌────────┐
#SH|│Client  │ │Docker  │   │LLM/STT │   │ Other  │
#MZ|│Browser │ │Host    │   │Server  │   │Devices │
#RY|│:3000   │ │:142    │   │${EXTERNAL_IP} │   │        │
#TS|└────────┘ └────┬───┘   └────────┘   └────────┘
#SB|                │
#YR|         ┌──────┴──────┐
#TT|         │             │
#HY|    ┌────▼────┐   ┌───▼────┐
#VP|    │SurrealDB│   │Open    │
#JS|    │:8000    │   │Notebook│
#HZ|    │         │   │:3000   │
#WJ|    └─────────┘   └────────┘
#YZ|```
#RK|
#WT|---
#MP|
#TQ|## Future Enhancements
#ZM|
#VJ|### Circuit Breaker Pattern
#ZP|
#PB|**Purpose**: Implement graceful degradation when external services (LLM/STT) become unavailable
#BS|
#PT|**Implementation Approach:**
#BS|1. **Detection**: Monitor external service health endpoints
#BS|2. **Fallback**: Queue requests or return cached responses
#BS|3. **Recovery**: Automatic retry with exponential backoff
#BS|4. **Alerting**: Notify administrators of service degradation
#BS|
#PT|**Benefits:**
#BS|- Prevents cascading failures
#BS|- Improves user experience during outages
#BS|- Provides time for troubleshooting without urgency
#BS|- Maintains core functionality (note creation, storage)
#BS|
#PT|**Priority**: Medium - Recommended for production deployments
#BS|
#HB|---
#JQ|
#NH|**Document Version:** 1.1  
#RT|**Last Updated:** 2026-06-14  
#VT|**Status:** Ready for Implementation  
#YY|**Next Review:** After initial deployment

(End of file - total 1738 lines)
