# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AdGuardHome Sync is a Go application that synchronizes configuration from one AdGuardHome instance to one or more replica instances. It supports syncing various features including filters, rewrites, services, clients, DNS/DHCP configs, and themes.

The application can run as:
- One-time sync: `adguardhome-sync run`
- Daemon mode with cron scheduling: `adguardhome-sync run --cron "0 */2 * * *"`
- With API server for remote triggering and metrics (default port 8080)

## Development Commands

### Build and Run
```bash
# Build the application
go build -o adguardhome-sync

# Run directly
go run main.go run

# Run with config file
go run main.go run --config path/to/config.yaml
```

### Testing
```bash
# Run all tests (includes code generation, linting, and test execution)
make test

# Run tests only (CI mode - no linting)
make test-ci

# Run tests for a specific package
go run bin/ginkgo run ./internal/client/...

# Run single test by name (using Ginkgo focus)
# Edit the test file and add 'F' prefix (FDescribe, FContext, FIt) to focus on specific tests
# Example: Change "It(...)" to "FIt(...)" then run:
make test-ci

# Fuzz testing
make fuzz
```

### Code Quality
```bash
# Run linter (auto-fixes issues)
make lint

# Format code (done automatically by lint)
# Uses gci, gofmt, gofumpt, goimports, golines with specific settings

# Tidy dependencies
make tidy
```

### Code Generation
```bash
# Generate all (model + mocks + deepcopy)
make generate

# Generate model from AdGuardHome OpenAPI schema
make model

# Generate mocks (uses mockgen)
make mocks

# Generate deepcopy methods (uses controller-gen)
make deepcopy-gen

# Update documentation
make docs
```

### Release
```bash
# Create and push release tag
make release

# Test release locally (without publishing)
make test-release
```

### Docker Testing
```bash
# Start test replica instances
make start-replica   # runs on port 9091
make start-replica2  # runs on port 9093

# Copy replica config for inspection
make copy-replica-config
```

## Code Architecture

### Core Packages

**`cmd/`** - CLI command definitions using Cobra
- `root.go`: Base command setup
- `run.go`: Main sync command with all feature flags

**`internal/client/`** - AdGuardHome API client
- `client.go`: HTTP client implementation using resty
- `client-methods.go`: API method wrappers (DHCP, DNS, filters, rewrites, etc.)
- `model/`: Auto-generated types from AdGuardHome OpenAPI spec
  - `model_generated.go`: Generated from OpenAPI (DO NOT EDIT MANUALLY)
  - `model-functions.go`: Custom helper methods (Clone, Equals, etc.)
  - `client/`: Client interfaces and implementations

**`internal/sync/`** - Synchronization engine
- `sync.go`: Main worker that orchestrates sync operations, cron scheduling, API server
- `action.go`: Defines sync actions for each feature
- `action-general.go`: Feature-specific sync implementations
- `http.go`: HTTP handlers for API server (trigger sync, get status)
- `scrape.go`: Metrics scraping from AdGuardHome instances

**`internal/config/`** - Configuration management
- `config.go`: Loads config from file, env vars, and flags (priority: flags > env > file)
- `env.go`: Environment variable parsing with `REPLICA#_URL` pattern support
- `file.go`: YAML config file parsing
- `flags.go`: Cobra flag definitions
- `validate.go`: JSON schema validation

**`internal/types/`** - Core data structures
- `types.go`: Config, AdGuardInstance, API, Features structs
- `features.go`: Feature toggle definitions
- `zz_generated.deepcopy.go`: Auto-generated deepcopy methods (DO NOT EDIT)

**`internal/metrics/`** - Prometheus metrics
- `metrics.go`: Metric definitions and collection
- `stats.go`: AdGuardHome stats processing

**`internal/versions/`** - Version compatibility checking between instances

**`internal/utils/`** - Utility functions (cloning, JSON comparison, pointers)

### Configuration Priority

Configuration is loaded in this order (later overrides earlier):
1. YAML config file (default: `$HOME/.adguardhome-sync.yaml`)
2. Environment variables (prefixed: `ORIGIN_`, `REPLICA#_`, `FEATURES_`, `API_`)
3. Command-line flags

### Replica Configuration Patterns

The app supports two patterns for defining replicas:
1. Single replica: `REPLICA_URL`, `REPLICA_USERNAME`, etc.
2. Multiple replicas: `REPLICA1_URL`, `REPLICA2_URL`, etc. (indexed from 1)

Both can be used simultaneously - they are deduplicated by URL+APIPath combination.

### Testing Framework

Uses **Ginkgo** (BDD framework) with **Gomega** (matchers):
- Test files: `*_test.go`
- Test suites: `*_suite_test.go` (sets up Ginkgo)
- Structure: `Describe` → `Context` → `It` blocks
- Mocking: Uses `go.uber.org/mock` for interface mocks

### Key Patterns

**Client Interface**: All AdGuardHome API interactions go through the `Client` interface, making it mockable for testing.

**Feature Toggles**: Each sync feature (DHCP, DNS, filters, etc.) can be independently enabled/disabled via config.

**Error Handling**: `continueOnError` flag allows sync to proceed even if individual features fail.

**AutoSetup**: Replicas can be automatically initialized if not yet configured (replaces AdGuardHome setup wizard).

## Important Notes

- The OpenAPI model (`internal/client/model/model_generated.go`) is generated from AdGuardHome's schema - never edit manually
- Deepcopy methods in `zz_generated.deepcopy.go` are auto-generated - never edit manually
- The project uses strict linting rules (see `.golangci.yaml`) - run `make lint` before committing
- Maximum line length is 128 characters (enforced by golines)
- Import order: standard → external → `github.com/bakito/adguardhome-sync` (enforced by gci)
- AdGuardHome version is tracked in `Makefile` (renovate keeps it updated)

## Environment Variables

See README.md for full list. Key ones:
- `LOG_LEVEL`: debug|info|warn|error (default: info)
- `LOG_FORMAT`: console|json (default: console)
- `ORIGIN_URL`, `ORIGIN_USERNAME`, `ORIGIN_PASSWORD`: Origin instance credentials
- `REPLICA#_URL`, `REPLICA#_USERNAME`, `REPLICA#_PASSWORD`: Replica credentials
- `CRON`: Cron expression for daemon mode
- `API_PORT`: API server port (0 disables, default: 8080)
- `FEATURES_*`: Individual feature toggles

## Production Deployment Guide

### Prerequisites

1. **Go Installation** (if building from source)
   ```bash
   # Download and install Go (tested with 1.23.4)
   wget https://go.dev/dl/go1.23.4.linux-amd64.tar.gz
   sudo tar -C /usr/local -xzf go1.23.4.linux-amd64.tar.gz
   export PATH=$PATH:/usr/local/go/bin
   ```

2. **Build from Source**
   ```bash
   go build -o adguardhome-sync
   ```

### Systemd Service Setup

1. **Create Service File** (`/etc/systemd/system/adguardhome-sync.service`)
   ```ini
   [Unit]
   Description=AdGuard Home Sync
   After=network.target

   [Service]
   Type=simple
   User=ansible  # or your preferred user
   WorkingDirectory=/path/to/adguardhome-sync
   Environment="PATH=/usr/local/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
   ExecStart=/path/to/adguardhome-sync/adguardhome-sync run --continueOnError
   Restart=always
   RestartSec=10

   [Install]
   WantedBy=multi-user.target
   ```

2. **Enable and Start Service**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable adguardhome-sync
   sudo systemctl start adguardhome-sync
   ```

### Common Production Issues and Solutions

#### Password Authentication Issues
**Problem**: Authentication fails with 401 Unauthorized despite correct credentials.

**Common Causes**:
1. **Special characters in password**: If your password contains special characters (especially `?`, `@`, `!`, etc.), they must be properly escaped in the config file:
   ```yaml
   origin:
     password: "PbHED5b5i@C?"  # Note the quotes for special characters
   ```

2. **Testing authentication**: Before configuring sync, test credentials directly:
   ```bash
   # Test login endpoint
   curl -X POST http://your-server:port/control/login \
     -H "Content-Type: application/json" \
     -d '{"name":"admin","password":"your-password"}'
   # Should return "OK" if successful
   ```

#### Network and Firewall Issues
**Problem**: Connection refused or timeout errors.

**Solutions**:
- Verify AdGuard Home API port (typically 80 or 3000)
- Check firewall rules between sync server and AdGuard instances
- For Sophos or similar firewalls, may need to whitelist dl.google.com for Go downloads

#### Configuration File vs Command Line
**Recommendation**: Use configuration file instead of command-line flags for production:
- Easier to manage complex passwords with special characters
- More maintainable for multiple replicas
- Cleaner systemd service definition

### Monitoring and Maintenance

#### Service Management Commands
```bash
# Check status
sudo systemctl status adguardhome-sync

# View logs
sudo journalctl -u adguardhome-sync -f

# Restart service
sudo systemctl restart adguardhome-sync
```

#### Manual Sync Trigger
```bash
# Trigger sync via API (if API port is enabled)
curl -X POST http://localhost:8080/api/v1/sync
```

#### Log Analysis
```bash
# Check for sync completion
sudo journalctl -u adguardhome-sync | grep "Sync done"

# Find errors
sudo journalctl -u adguardhome-sync | grep ERROR

# Monitor sync frequency
sudo journalctl -u adguardhome-sync --since "1 hour ago" | grep "Start sync"
```

### Security Considerations

1. **Config File Permissions**
   ```bash
   chmod 600 ~/.adguardhome-sync.yaml  # Restrict access to config with passwords
   ```

2. **API Server Security**
   - Consider disabling API port (`API_PORT: 0`) if not needed
   - Or restrict access via firewall rules if API is required

3. **Password Management**
   - Consider using environment variables for sensitive data in production
   - Or use a secrets management system for enhanced security

### Performance Notes

- Typical sync duration: 1-2 seconds
- Memory usage: ~6-10 MB
- CPU usage: Minimal (<1% average)
- Network impact: Negligible for internal networks

### Deployment Checklist

- [ ] Go installed (if building from source)
- [ ] Binary built and tested
- [ ] Configuration file created with correct credentials
- [ ] Password special characters properly escaped
- [ ] Initial manual sync tested successfully
- [ ] Systemd service created and enabled
- [ ] Service started and logs checked
- [ ] Sync schedule verified (check cron expression)
- [ ] API endpoints tested (if enabled)
- [ ] Monitoring/alerting configured (optional)
