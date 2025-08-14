# Prometheus CVMFS Exporter

A Prometheus exporter for monitoring CernVM File System (CVMFS) clients. This tool collects comprehensive metrics from CVMFS repositories and exposes them in Prometheus format for monitoring and alerting.

Other exporters for CVMFS exist - this one was written to have a lightweight package without dependencies (in particular python). If you don't mind python, you could also take a look at https://github.com/guilbaults/cvmfs-exporter. Thanks to Simon Guilbaults for inspiration of this exporter, and having reserved port 9868 for CVMFS!

## Overview

The prometheus-cvmfs-exporter provides detailed insights into CVMFS client performance, cache utilization, network activity, and system resource usage. It supports both standalone execution and systemd service deployment with socket activation.

### Features

- **Comprehensive Metrics**: Collects 20+ different CVMFS metrics per repository
- **Multi-Repository Support**: Automatically discovers and monitors all mounted CVMFS repositories

## Installation

### Package Installation

**RPM-based systems (RHEL, CentOS, AlmaLinux, Fedora):**
```bash
sudo rpm -ivh prometheus-cvmfs-exporter-1.0.0-1.el9.noarch.rpm
sudo systemctl enable --now cvmfs-client-prometheus.socket
```

**DEB-based systems (Debian, Ubuntu):**
```bash
sudo dpkg -i prometheus-cvmfs-exporter_1.0.0-1_all.deb
sudo systemctl enable --now cvmfs-client-prometheus.socket
```

### Manual Installation

```bash
# Install script and systemd files
make install install-systemd

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable --now cvmfs-client-prometheus.socket
```

## Usage

### Command Line Options

```bash
/usr/libexec/cvmfs/cvmfs-prometheus.sh [OPTIONS]

Options:
  -h, --help                    Show help message
  --http                        Add HTTP protocol header to output
  --non-standard-mountpoints    Use cvmfs_config status instead of findmnt
                               for repository discovery
```

### Usage Examples

**Direct execution (one-time metrics collection):**
```bash
/usr/libexec/cvmfs/cvmfs-prometheus.sh
```

**HTTP server mode (for web scraping):**
```bash
/usr/libexec/cvmfs/cvmfs-prometheus.sh --http
```

**Non-standard mountpoints:**
```bash
/usr/libexec/cvmfs/cvmfs-prometheus.sh --non-standard-mountpoints
```

### Systemd Service

The exporter runs as a systemd socket-activated service:

```bash
# Check service status
sudo systemctl status cvmfs-client-prometheus.socket

# View service logs
sudo journalctl -u cvmfs-client-prometheus@.service

# Test metrics endpoint
curl http://localhost:9868
```

## Metrics Summary

The exporter collects the following categories of metrics with consistent naming:

### Cache Metrics (`cvmfs_cache_*`)
- `cvmfs_cache_cached_bytes` - Currently cached data size
- `cvmfs_cache_pinned_bytes` - Pinned cache data size
- `cvmfs_cache_total_size_bytes` - Configured cache limit
- `cvmfs_cache_physical_size_bytes` - Physical cache volume size
- `cvmfs_cache_physical_avail_bytes` - Available cache space
- `cvmfs_cache_hitrate` - Cache hit rate percentage
- `cvmfs_cache_ncleanup24` - Cache cleanups in last 24 hours
- `cvmfs_cache_mode` - Cache mode (0=unknown, 1=read-write, 2=read-only)

### Network & Download Metrics (`cvmfs_net_*`)
- `cvmfs_net_rx_total` - Total bytes downloaded since mount
- `cvmfs_net_ndownload_total` - Total files downloaded since mount
- `cvmfs_net_speed` - Average download speed
- `cvmfs_net_proxy` - Available proxy servers
- `cvmfs_net_active_proxy` - Currently active proxy
- `cvmfs_net_timeout` - Proxy connection timeout
- `cvmfs_net_timeout_direct` - Direct connection timeout

### Repository Status Metrics (`cvmfs_repo_*`)
- `cvmfs_repo` - Repository version and revision information
- `cvmfs_repo_uptime_seconds` - Time since repository mount
- `cvmfs_repo_mount_epoch_timestamp` - Repository mount timestamp
- `cvmfs_repo_expires_seconds` - Root catalog expiration time
- `cvmfs_repo_version` - Numeric repository version for easier querying
- `cvmfs_repo_revision` - Repository revision number
- `cvmfs_repo_nclg` - Number of loaded nested catalogs

### System Resource Metrics (`cvmfs_sys_*`)
- `cvmfs_sys_cpu_user_total` - CPU time in userspace
- `cvmfs_sys_cpu_system_total` - CPU time in kernel space
- `cvmfs_sys_maxfd` - Maximum file descriptors available
- `cvmfs_sys_usedfd` - Currently used file descriptors
- `cvmfs_sys_useddirp` - File descriptors issued to clients
- `cvmfs_sys_ndiropen` - Number of open directories
- `cvmfs_sys_pid` - CVMFS process ID
- `cvmfs_sys_inode_max` - Highest possible inode number
- `cvmfs_sys_memory_usage_bytes` - CVMFS process memory usage
- `cvmfs_sys_nioerr_total` - Total I/O errors encountered
- `cvmfs_sys_timestamp_last_ioerr` - Timestamp of last I/O error
- `cvmfs_sys_drainout_mode` - Drainout mode status
- `cvmfs_sys_maintenance_mode` - Maintenance mode status
- `cvmfs_sys_nfs_mode` - NFS mode enabled status

### Internal Affairs Metrics (`cvmfs_internal_*`)
- `cvmfs_internal_pathstring_*` - PathString statistics
- `cvmfs_internal_namestring_*` - NameString statistics
- `cvmfs_internal_linkstring_*` - LinkString statistics
- `cvmfs_internal_inode_tracker_*` - Inode tracker statistics
- `cvmfs_internal_dentry_tracker_*` - Dentry tracker statistics
- `cvmfs_internal_page_cache_tracker_*` - Page cache tracker statistics
- `cvmfs_internal_sqlite_*` - SQLite internal statistics

## Version Compatibility

The script automatically detects the CVMFS version and adapts accordingly:

- **CVMFS 2.13.2+**: Uses the native `cvmfs_talk metrics prometheus` command with consistent metric naming
- **CVMFS 2.13.2 (exact)**: Applies postprocessing to rename metrics for consistency
- **Older versions**: Uses legacy extended attribute collection with consistent naming

## Configuration

### Repository Discovery

By default, the exporter uses `findmnt` to discover CVMFS repositories mounted under `/cvmfs`. For non-standard setups, use the `--non-standard-mountpoints` flag to use `cvmfs_config status` instead.

### Systemd Configuration

The systemd service includes security hardening and resource limits:
- Runs as `cvmfs` user/group
- Restricted system call access
- Memory limit: 32MB
- CPU weight: 30 (low priority)
- I/O scheduling: best-effort, priority 7

### Prometheus Configuration

Add the following to your Prometheus configuration:

```yaml
scrape_configs:
  - job_name: 'cvmfs-exporter'
    static_configs:
      - targets: ['localhost:9868']
    scrape_interval: 30s
    scrape_timeout: 10s
```

## Requirements

- **CVMFS**: CernVM File System client installed and configured
- **System Tools**: `attr`, `bc`, `findmnt`, `grep`, `cut`, `tr`
- **Permissions**: Read access to CVMFS cache files and extended attributes
- **Network**: Port 9868 for HTTP metrics endpoint (when using systemd socket)

## Building from Source

```bash
# Build packages
make package          # Build both RPM and DEB
make rpm             # Build RPM only
make deb             # Build DEB only (requires Debian/Ubuntu)

# Install from source
make install install-systemd
```

## Troubleshooting

### Common Issues

1. **Permission Denied**: Ensure the user has read access to CVMFS cache files
2. **No Metrics**: Verify CVMFS repositories are mounted and accessible
3. **Socket Connection Failed**: Check if systemd socket is active and port 9868 is available

### Debug Commands

```bash
# Test script manually
/usr/libexec/cvmfs/cvmfs-prometheus.sh --help

# Check CVMFS status
cvmfs_config status

# Verify repository mounts
findmnt -t fuse.cvmfs

# Check systemd service
systemctl status cvmfs-client-prometheus.socket
journalctl -u cvmfs-client-prometheus@.service
```

## License

This project is licensed under the BSD 3-Clause License. See the [LICENSE](LICENSE) file for details.

## Contributing

Contributions are welcome! Please feel free to submit issues, feature requests, or pull requests.

## Related Projects

- [CernVM-FS](https://github.com/cvmfs/cvmfs) - The CernVM File System
- [Prometheus](https://prometheus.io/) - Monitoring and alerting toolkit
