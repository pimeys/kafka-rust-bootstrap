# Kafka Testing Environment with rskafka Integration

A comprehensive Docker Compose setup for testing Apache Kafka features with multiple authentication methods, designed as a foundation for developing and testing Kafka applications using the `rskafka` Rust client library.

## Overview

This repository provides a unified Kafka testing environment that supports multiple authentication methods on different ports, along with a Rust application demonstrating `rskafka` client usage patterns. It's designed to be a reusable foundation for any Kafka project where you need to trial and test different Kafka features, authentication methods, and client configurations.

## Key Features

- **Unified Kafka Broker**: Single Kafka instance (KRaft mode, no Zookeeper) supporting multiple authentication methods
- **Multiple Auth Methods**: PLAINTEXT, SASL/PLAIN, SASL/SCRAM, SSL/TLS, and mTLS on different ports
- **rskafka Integration**: Production-ready Rust client examples with comprehensive error handling
- **Partition Management**: Smart partition calculation using Murmur2 hash algorithm
- **SSL/TLS Ready**: Complete certificate generation and management
- **Development Friendly**: Easy setup with automatic topic creation and health checks

## Architecture

### Kafka Broker Configuration

The setup uses a single Kafka broker running in KRaft mode (no Zookeeper) with multiple listeners:

| Authentication | Port | Protocol | Use Case |
|----------------|------|----------|----------|
| None (PLAINTEXT) | 9092 | PLAINTEXT | Development, testing |
| SASL/PLAIN | 9093 | SASL_PLAINTEXT | Username/password auth |
| SASL/SCRAM | 9094 | SASL_PLAINTEXT | Secure username/password |
| SSL/TLS | 9095 | SSL | Encrypted transport |
| mTLS | 9096 | SSL | Mutual certificate auth |

### Rust Application Structure

The Rust application (`src/main.rs`) demonstrates:

- **Multi-broker support**: Seamless switching between authentication methods
- **Producer patterns**: Round-robin partitioning with configurable strategies
- **Error handling**: Comprehensive error management for production use
- **SSL/TLS support**: Full certificate-based authentication
- **Partition management**: Automatic partition discovery and smart routing

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Rust (for running the test application)
- OpenSSL (for SSL certificate generation)

### Setup

1. **Generate SSL Certificates**:
   ```bash
   cd config
   chmod +x generate_ssl_certs.sh
   ./generate_ssl_certs.sh
   ```

2. **Start Kafka Environment**:
   ```bash
   docker compose up -d
   ```

3. **Wait for Full Initialization**:
   ```bash
   docker compose logs kafka-topics
   # Wait for "All topic operations completed successfully"
   ```

4. **Run Rust Tests**:
   ```bash
   cargo run
   ```

### Verification

Check that all services are healthy:
```bash
docker compose ps
```

Test connectivity to all authentication methods:
```bash
# PLAINTEXT
docker compose exec kafka kafka-topics --bootstrap-server localhost:9092 --list

# SASL/PLAIN
docker compose exec kafka kafka-console-consumer --bootstrap-server localhost:9093 \
  --consumer.config /tmp/sasl_plain_client.properties --topic test-topic-1 --max-messages 1

# SSL/TLS
docker compose exec kafka kafka-topics --bootstrap-server localhost:9095 \
  --command-config /tmp/ssl_client.properties --list
```

## Authentication Methods

### 1. PLAINTEXT (Port 9092)
- **Use**: Development and testing
- **Connection**: `localhost:9092`
- **Security**: None
- **rskafka config**:
  ```rust
  ClientBuilder::new(vec!["localhost:9092".to_string()])
  ```

### 2. SASL/PLAIN (Port 9093)
- **Use**: Basic username/password authentication
- **Connection**: `localhost:9093`
- **Credentials**:
  - `admin` / `admin-secret`
  - `testuser` / `testuser-secret`
- **rskafka config**:
  ```rust
  ClientBuilder::new(vec!["localhost:9093".to_string()])
      .sasl_config(SaslConfig::Plain {
          username: "admin".to_string(),
          password: "admin-secret".to_string(),
      })
  ```

### 3. SASL/SCRAM (Port 9094)
- **Use**: Secure password authentication with challenge-response
- **Connection**: `localhost:9094`
- **Mechanism**: SCRAM-SHA-512
- **Credentials**: Same as SASL/PLAIN
- **rskafka config**:
  ```rust
  ClientBuilder::new(vec!["localhost:9094".to_string()])
      .sasl_config(SaslConfig::ScramSha512 {
          username: "admin".to_string(),
          password: "admin-secret".to_string(),
      })
  ```

### 4. SSL/TLS (Port 9095)
- **Use**: Encrypted transport, server authentication
- **Connection**: `localhost:9095`
- **Certificates**: Auto-generated in `config/ssl/`
- **Client auth**: None required

### 5. mTLS (Port 9096)
- **Use**: Mutual certificate authentication
- **Connection**: `localhost:9096`
- **Certificates**: Client and server certificates required
- **Security**: Highest (mutual authentication + encryption)

## Rust Development with rskafka

### Project Structure

```
src/
├── main.rs              # Main application with examples
└── ...                  # Add your modules here

config/
├── ssl/                 # SSL certificates
├── *.conf              # Kafka client configurations
└── generate_ssl_certs.sh

kraft-configs/           # Kafka server configuration
compose.yml             # Docker services definition
```

### Key Components

#### KafkaProducer Implementation

The `KafkaProducer` struct provides:

- **Multi-auth support**: Seamlessly switch between authentication methods
- **Partition management**: Automatic partition discovery and balanced distribution
- **Error handling**: Production-ready error management
- **Performance**: Efficient round-robin partitioning with Murmur2 hashing

#### Usage Patterns

```rust
// Create producer for any authentication method
let config = KafkaConfig {
    name: "test-producer".to_string(),
    bootstrap_servers: "localhost:9092".to_string(),
    auth_type: AuthType::Plain,
};

let mut producer = KafkaProducer::new(config);
producer.initialize().await?;

// Send messages with automatic partitioning
producer.produce_message("test-topic-1", "key1", "Hello Kafka!").await?;
```

### Adding New Features

This foundation makes it easy to:

1. **Test new authentication methods**: Add to `AuthType` enum
2. **Implement new producers/consumers**: Extend the client creation logic
3. **Test different serialization**: Modify message encoding
4. **Benchmark performance**: Add timing metrics to existing functions
5. **Test failure scenarios**: Use different broker configurations

## Development Workflow

### Testing New Features

1. **Modify Rust code** in `src/main.rs` or add new modules
2. **Update Kafka configuration** if needed in `kraft-configs/`
3. **Test with specific auth method**:
   ```bash
   # Test only SASL/SCRAM
   cargo run | grep "kafka-sasl-scram"
   ```
4. **Validate across all methods**:
   ```bash
   cargo run
   ```

### Adding New Authentication Methods

1. Add new variant to `AuthType` enum
2. Implement client creation in `create_client()`
3. Add broker configuration to `compose.yml`
4. Update `create_kafka_configs()` function
5. Test the new configuration

### SSL Certificate Management

Certificates are automatically generated for:
- Server certificate (`kafka.keystore.jks`)
- Truststore (`kafka.truststore.jks`)
- Client certificates for mTLS

To regenerate certificates:
```bash
cd config
rm -rf ssl/
./generate_ssl_certs.sh
docker compose restart kafka
```

## Monitoring and Debugging

### Health Checks

The setup includes comprehensive health checks:
```bash
# Check all services
docker compose ps

# Check specific service logs
docker compose logs kafka
docker compose logs kafka-scram-users
docker compose logs kafka-topics
```

### Connection Testing

Test each authentication method:
```bash
# Test with Kafka tools
docker compose exec kafka kafka-broker-api-versions --bootstrap-server localhost:9092
docker compose exec kafka kafka-broker-api-versions --bootstrap-server localhost:9093 \
  --command-config /tmp/sasl_plain_client.properties

# Test with Rust application
RUST_LOG=debug cargo run
```

### Common Issues

1. **SSL Connection Failures**: Regenerate certificates
2. **SCRAM Authentication Errors**: Check if users were created successfully
3. **Topic Not Found**: Verify topic creation completed
4. **Connection Timeouts**: Ensure all health checks pass

## Performance Testing

The included producer supports performance testing:

```rust
// Batch message production
for i in 0..1000 {
    producer.produce_message(
        "test-topic-1",
        &format!("key-{}", i),
        &format!("Message {}", i)
    ).await?;
}
```

Monitor performance with:
```bash
# Producer metrics
docker compose exec kafka kafka-run-class kafka.tools.JmxTool \
  --object-name kafka.producer:type=producer-metrics,client-id=* \
  --jmx-url service:jmx:rmi:///jndi/rmi://localhost:9999/jmxrmi

# Topic metrics
docker compose exec kafka kafka-log-dirs --bootstrap-server localhost:9092 \
  --topic-list test-topic-1 --describe
```

## Cleanup

### Partial Cleanup
```bash
# Stop services but keep data
docker compose stop

# Restart services
docker compose start
```

### Full Cleanup
```bash
# Remove all containers and data
docker compose down -v

# Clean up images (optional)
docker system prune
```

## Extending for Production

This testing environment provides patterns for:

- **Configuration management**: Environment-based auth selection
- **Error handling**: Comprehensive error types and recovery
- **Connection pooling**: Client reuse patterns
- **Monitoring integration**: Metrics and logging hooks
- **Security**: Certificate management and credential handling

Use this foundation to build production Kafka applications with confidence in your authentication and connection handling.
