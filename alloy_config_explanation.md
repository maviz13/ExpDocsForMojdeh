# alloy config explanation

## table of content(components)
[DOCKER CONTAINER DISCOVERY](#docker-container-discovery)

[DOCKER CONSOLE LOG PROCESSING PIPELINE](#docker-console-log-processing-pipeline)

[DOCKER LOG RELABELING RULES](#docker-log-relabeling-rules)

[DOCKER LOG COLLECTION](#docker-log-collection)

[STORE SERVICE FILE LOG RELABELING](#store-service-file-log-relabeling)

[STORE SERVICE FILE MATCHING](#store-service-file-matching)

[STORE SERVICE FILE LOG PROCESSING](#store-service-file-log-processing)

[STORE SERVICE FILE LOG COLLECTION](#store-service-file-log-collection)

[SYNC SERVICE FILE LOG RELABELING](#sync-service-file-log-relabeling)

[SYNC SERVICE FILE MATCHING](#sync-service-file-matching)

[SYNC SERVICE FILE LOG PROCESSING](#sync-service-file-log-processing)

[SYNC SERVICE FILE LOG COLLECTION](#sync-service-file-log-collection)

[LOKI OUTPUT CONFIGURATION](#loki-output-configuration)

[Logging configuration](#logging-configuration)

[Host metrics exporter using Unix system metrics](#host-metrics-exporter-using-unix-system-metrics)

[Scrape configuration for host metrics](#scrape-configuration-for-host-metrics)

[Alloy's own metrics exporter](#alloys-own-metrics-exporter)

[Scrape configuration for Alloy metrics](#scrape-configuration-for-alloy-metrics)

[Remote write configuration](#remote-write-configuration)



### DOCKER CONTAINER DISCOVERY

    
    // This component discovers Docker containers running on the host
    discovery.docker "docker_console" {
            // Connect to the Docker daemon socket
            host             = "unix:///var/run/docker.sock"
            // How often to check for new/removed containers (10 seconds)
            refresh_interval = "10s"
    }
    

###  DOCKER CONSOLE LOG PROCESSING PIPELINE

    
    // Creates a processing pipeline specifically for Docker container stdout/stderr logs
    loki.process "docker_console" {
            // Where to send processed logs (to the default Loki writer)
            forward_to = [loki.write.default.receiver]

            // STAGE 1: Parse Docker-specific log format
            // Extracts container ID, name, and other Docker metadata from log lines
            stage.docker { }

            // STAGE 2: Remove ANSI color codes
            // Uses a raw string literal (backticks) to avoid escape sequence issues
            // Regex matches: \u001B[ followed by color codes (e.g., 31m, 1;32m, etc.)
            stage.replace {
                    expression = `\\u001B\\[[0-9;]*[A-Za-z]`
            }

            // STAGE 3: Extract log level from message content
            // Case-insensitive regex to find error, warn, info, debug, trace, fatal
            // Creates a 'level' label based on what's found in the message
            stage.regex {
                    expression = "(?i)(?P<level>error|warn|info|debug|trace|fatal)"
                    source     = "message"  // Search in the message field
            }

            // STAGE 4: Final output formatting
            // Ensures the processed message is what gets sent to Loki
            stage.output {
                    source = "message"
            }
    }
    


### DOCKER LOG RELABELING RULES

    
    // Applies labels to discovered Docker containers for better log organization
    discovery.relabel "docker_console" {
            targets = []  // Empty initially, will be populated by discovery.docker

            // RULE 1: Extract service name from container name
            // Matches 'store_s' or 'sync_s' and creates a 'service' label
            rule {
                    source_labels = ["__meta_docker_container_name"]
                    regex         = "/?(store_s|sync_s)"
                    target_label  = "service"
            }

            // RULE 2: Extract clean container name (without leading slash)
            // Creates a 'container' label with just the container name
            rule {
                    source_labels = ["__meta_docker_container_name"]
                    regex         = "/?(.*)"
                    target_label  = "container"
            }

            // RULE 3: Identify log source type
            // All logs from this pipeline are from console output
            rule {
                    target_label = "log_file_type"
                    replacement  = "console"
            }

            // RULE 4: Set job identifier
            // Helps identify which collection pipeline this came from
            rule {
                    target_label = "job"
                    replacement  = "docker_console"
            }
    }
    

### DOCKER LOG COLLECTION

    
    // Actually collects logs from discovered Docker containers
    loki.source.docker "docker_console" {
            // Docker connection details
            host             = "unix:///var/run/docker.sock"
            // Which containers to monitor (from discovery component)
            targets          = discovery.docker.docker_console.targets
            // Where to send raw logs for processing
            forward_to       = [loki.process.docker_console.receiver]
            // Labeling rules to apply (from discovery.relabel component)
            relabel_rules    = discovery.relabel.docker_console.rules
            // How often to check for container changes
            refresh_interval = "10s"
    }
    


### STORE SERVICE FILE LOG RELABELING

    
    // Static configuration for store service file logs (not Docker stdout)
    discovery.relabel "store_file" {
            // Manual target definition for file-based logs
            targets = [{
                    __address__   = "localhost",           // Not used for files, but required
                    __path__      = "/app/logs/store/*.log", // File pattern to monitor
                    container     = "file",                // Identify as file log (not container)
                    log_file_type = "file",                // Log source type
                    service       = "store_s",             // Which service these logs belong to
            }]

            // RULE 1: Extract filename from full path
            // Creates 'filename' label with just the log filename (e.g., error.log)
            rule {
                    source_labels = ["__path__"]          // Source is the file path
                    regex         = ".*/(.*\\.log)$"      // Capture the filename part
                    target_label  = "filename"            // Store as 'filename' label
            }

            // RULE 2: Set job identifier for file collection
            rule {
                    target_label = "job"
                    replacement  = "store_file"           // Identifies this file collection job
            }
    }
    


### STORE SERVICE FILE MATCHING

    
    // Converts the static targets into a format suitable for file monitoring
    local.file_match "store_file" {
            // Get the formatted targets from the relabeling component
            path_targets = discovery.relabel.store_file.output
    }
    

### STORE SERVICE FILE LOG PROCESSING

    
    // Processing pipeline for JSON-formatted file logs from store service
    loki.process "store_file" {
            // Send processed logs to Loki
            forward_to = [loki.write.default.receiver]

            // STAGE 1: Parse JSON log structure
            // Extracts fields from JSON log format: {"level": "error", "message": "...", "timestamp": "..."}
            stage.json {
                    expressions = {
                            level     = "level",      // Extract 'level' field
                            message   = "message",    // Extract 'message' field  
                            timestamp = "timestamp",  // Extract 'timestamp' field
                    }
            }

            // STAGE 2: Use custom timestamp from JSON (not file modification time)
            // Parses timestamps in format: YY-MM-DD HH:MM:SS.mmm (25-09-08 11:31:13.501)
            stage.timestamp {
                    source = "timestamp"              // Use the extracted timestamp field
                    format = "06-01-02 15:04:05.000" // Go reference time format: YY-MM-DD HH:MM:SS.mmm
            }

            // STAGE 3: Create label from log level
            // Adds 'level' as a label for easier filtering in Grafana
            stage.labels {
                    values = {
                            level = "level",          // Create label from the extracted level field
                    }
            }

            // STAGE 4: Final output
            stage.output {
                    source = "message"                // Send the processed message to Loki
            }
    }
    

### STORE SERVICE FILE LOG COLLECTION

    
    // Monitors and collects logs from store service log files
    loki.source.file "store_file" {
            // Which files to monitor (from file_match component)
            targets               = local.file_match.store_file.targets
            // Where to send logs for processing
            forward_to            = [loki.process.store_file.receiver]
            // Tracks file read position across restarts (legacy Promtail compatibility)
            legacy_positions_file = "/var/lib/promtail/positions.yaml"
    }
    


### SYNC SERVICE FILE LOG RELABELING

    
    // Static configuration for sync service file logs (mirrors store service setup)
    discovery.relabel "sync_file" {
            targets = [{
                    __address__   = "localhost",
                    __path__      = "/app/logs/sync/*.log",  // Different path for sync service
                    container     = "file",
                    log_file_type = "file",
                    service       = "sync_s",                 // Different service name
            }]

            rule {
                    source_labels = ["__path__"]
                    regex         = ".*/(.*\\.log)$"
                    target_label  = "filename"
            }

            rule {
                    target_label = "job"
                    replacement  = "sync_file"                // Different job name
            }
    }
    


### SYNC SERVICE FILE MATCHING


    
    local.file_match "sync_file" {
            path_targets = discovery.relabel.sync_file.output
    }
    

### SYNC SERVICE FILE LOG PROCESSING

    
    // Same processing pipeline as store service (JSON format, same timestamp format)
    loki.process "sync_file" {
            forward_to = [loki.write.default.receiver]

            stage.json {
                    expressions = {
                            level     = "level",
                            message   = "message",
                            timestamp = "timestamp",
                    }
            }

            stage.timestamp {
                    source = "timestamp"
                    format = "06-01-02 15:04:05.000"
            }

            stage.labels {
                    values = {
                            level = "level",
                    }
            }

            stage.output {
                    source = "message"
            }
    }
    

### SYNC SERVICE FILE LOG COLLECTION

    
    loki.source.file "sync_file" {
            targets               = local.file_match.sync_file.targets
            forward_to            = [loki.process.sync_file.receiver]
            legacy_positions_file = "/var/lib/promtail/positions.yaml"
    }
    

### LOKI OUTPUT CONFIGURATION

    
    // Defines where to send all processed logs (the final destination)
        loki.write "default" {
                endpoint {
                        // Loki API endpoint (service name from Docker Compose)
                        url       = "http://loki:3100/loki/api/v1/push"
                        // Multi-tenant identifier (must match Loki config)
                        tenant_id = "mojdeh"
                }
                // No additional labels to add to all log entries
                external_labels = {}
        }




### Logging configuration

    logging {
    level  = "info"        // Sets the logging level to "info". Options: debug, info, warn, error. "info" gives general operational logs.
    format = "logfmt"      // Sets the log format to logfmt, which outputs logs as key=value pairs, easily readable by machines.
    }

### Host metrics exporter using Unix system metrics

    prometheus.exporter.unix "host_metrics" { 
    // "host_metrics" is the name of this exporter instance

    set_collectors = [    
        // List of system metrics collectors to enable for this host
        "cpufreq",        // CPU frequency info
        "diskstats",      // Disk I/O statistics
        "filefd",         // File descriptor usage
        "filesystem",     // Filesystem usage stats
        "loadavg",        // System load averages
        "meminfo",        // Memory info (total, free, used, buffers, etc.)
        "netdev",         // Network interface statistics
        "netstat",        // Network protocol statistics (TCP, UDP, etc.)
        "stat",           // Basic system statistics (context switches, interrupts)
        "time",           // System time metrics
        "vmstat",         // Virtual memory statistics
    ]

    rootfs_path = "/host"    
    // Path to the host filesystem inside the container.
    // Needed to properly read /proc, /sys, and other host files.
    }

### Scrape configuration for host metrics

    prometheus.scrape "host_metrics" { 
    targets    = prometheus.exporter.unix.host_metrics.targets 
    // Uses the targets generated by the "host_metrics" exporter
    forward_to = [prometheus.remote_write.default.receiver]  
    // Sends scraped metrics to the configured remote write receiver (Prometheus)
    job_name   = "alloy-host-metrics"  
    // The job name under which metrics will appear in Prometheus
    }

### Alloy's own metrics exporter

    prometheus.exporter.self "alloy_self" { } 
    // Collects metrics about the Alloy process itself (CPU, memory, internal stats)
    // No additional configuration is needed; default targets point to Alloy

### Scrape configuration for Alloy metrics

    prometheus.scrape "alloy_self" {
    targets    = prometheus.exporter.self.alloy_self.targets
    // Uses Alloy’s self-exporter targets
    forward_to = [prometheus.remote_write.default.receiver]
    // Sends Alloy's internal metrics to Prometheus remote write
    job_name   = "alloy"
    // Metrics will appear under job name "alloy" in Prometheus
    }


### Remote write configuration

    prometheus.remote_write "default" {
    endpoint {
        url = "http://prometheus:9090/api/v1/write"  
        // URL of the Prometheus instance to forward metrics to
    }
    }







