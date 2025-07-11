# TECHNICAL_IMPLEMENTATION.md

## Colloquium Technical Implementation Strategy

This document provides detailed technical implementation guidance for building Colloquium following the proven architectural patterns from Permuto and Computo. It emphasizes practical, phased development with rigorous testing and quality gates to avoid the over-engineering pitfalls common in stream processing platforms.

## Implementation Philosophy

### 1. Evolutionary Development Model
Following the successful Permuto → Computo evolution:

- **Start Minimal**: Single-node, in-memory processing with basic features
- **Prove Value**: Validate core use cases before adding complexity
- **Add Incrementally**: Each phase builds proven value on solid foundation
- **Maintain Simplicity**: Resist feature creep and maintain operational simplicity

### 2. Computo-First Integration
Leverage Computo's proven transformation engine:

- **Zero Reimplementation**: Use Computo library directly, no custom DSL
- **Performance Optimization**: Pre-compile Computo scripts for streaming workloads
- **Context Enhancement**: Inject stream metadata without modifying Computo core
- **Error Isolation**: Isolate transformation errors from stream processing engine

### 3. Quality-First Development
Apply rigorous engineering standards from Computo:

- **Test-Driven Development**: Write tests before implementation
- **Performance Benchmarking**: Measure performance at every phase
- **Documentation-Driven**: Document architecture decisions as they're made
- **Security by Design**: Build security into every component from the start

## Phase 1: Core Stream Processing Engine (Months 1-3)

### MVP Architecture

```cpp
// Core types following Computo's JSON-native approach
#include <nlohmann/json.hpp>
#include <chrono>
#include <string>
#include <optional>
#include <memory>
#include <thread>
#include <mutex>
#include <queue>

struct EventMetadata {
    std::string source;
    std::string pipeline;
    std::optional<std::string> routing_key;
    uint32_t retry_count = 0;
};

struct JsonEvent {
    std::string id;
    int64_t timestamp;
    nlohmann::json data;
    EventMetadata metadata;
    
    // Generate timestamp
    static int64_t current_timestamp() {
        return std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::system_clock::now().time_since_epoch()).count();
    }
};

// Main engine following single responsibility principle
class StreamEngine {
private:
    std::unique_ptr<EventRouter> event_router_;
    std::unique_ptr<ProcessingPipeline> processing_pipeline_;
    std::unique_ptr<StorageManager> storage_manager_;
    std::unique_ptr<MetricsCollector> metrics_collector_;
    std::atomic<bool> running_{false};
    
public:
    StreamEngine();
    ~StreamEngine() = default;
    
    void process_event(const JsonEvent& event);
    void start();
    void stop();
    bool is_running() const { return running_.load(); }
};
```

### Phase 1 Requirements (30 Core Features)

#### Event Ingestion and Routing
**REQ-001 to REQ-010**: Basic event processing foundation

```cpp
// Simple HTTP source for MVP using cpp-httplib
#include <httplib.h>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <random>

class HttpSource {
private:
    std::string host_;
    int port_;
    size_t max_body_size_;
    std::queue<JsonEvent> event_queue_;
    std::mutex queue_mutex_;
    std::condition_variable queue_cv_;
    std::unique_ptr<httplib::Server> server_;
    std::atomic<bool> running_{false};
    
public:
    HttpSource(const std::string& host, int port, size_t max_body_size)
        : host_(host), port_(port), max_body_size_(max_body_size) {}
    
    void start() {
        server_ = std::make_unique<httplib::Server>();
        running_ = true;
        
        server_->Post("/events", [this](const httplib::Request& req, httplib::Response& res) {
            return handle_event(req, res);
        });
        
        server_->Get("/health", [](const httplib::Request&, httplib::Response& res) {
            res.set_content("{\"status\":\"healthy\"}", "application/json");
        });
        
        server_->listen(host_, port_);
    }
    
    void stop() {
        running_ = false;
        if (server_) {
            server_->stop();
        }
    }
    
    std::optional<JsonEvent> get_next_event() {
        std::unique_lock<std::mutex> lock(queue_mutex_);
        queue_cv_.wait(lock, [this] { return !event_queue_.empty() || !running_; });
        
        if (event_queue_.empty()) {
            return std::nullopt;
        }
        
        JsonEvent event = std::move(event_queue_.front());
        event_queue_.pop();
        return event;
    }
    
private:
    bool handle_event(const httplib::Request& req, httplib::Response& res) {
        try {
            if (req.body.size() > max_body_size_) {
                res.status = 413; // Payload Too Large
                res.set_content("{\"error\":\"Request body too large\"}", "application/json");
                return false;
            }
            
            // Parse JSON, validate schema, generate event ID
            auto data = nlohmann::json::parse(req.body);
            
            JsonEvent event;
            event.id = generate_event_id();
            event.timestamp = JsonEvent::current_timestamp();
            event.data = std::move(data);
            event.metadata = EventMetadata{"http", "default", std::nullopt, 0};
            
            // Add to queue
            {
                std::lock_guard<std::mutex> lock(queue_mutex_);
                event_queue_.push(std::move(event));
            }
            queue_cv_.notify_one();
            
            res.status = 202; // Accepted
            res.set_content("{\"status\":\"accepted\"}", "application/json");
            return true;
            
        } catch (const nlohmann::json::exception& e) {
            res.status = 400;
            res.set_content("{\"error\":\"Invalid JSON\"}", "application/json");
            return false;
        } catch (const std::exception& e) {
            res.status = 500;
            res.set_content("{\"error\":\"Internal server error\"}", "application/json");
            return false;
        }
    }
    
    std::string generate_event_id() {
        static thread_local std::random_device rd;
        static thread_local std::mt19937 gen(rd());
        static thread_local std::uniform_int_distribution<> dis(0, 15);
        
        std::string uuid;
        uuid.reserve(32);
        for (int i = 0; i < 32; ++i) {
            uuid += "0123456789abcdef"[dis(gen)];
        }
        return uuid;
    }
};
```

#### Basic Event Storage
**REQ-006 to REQ-010**: Persistent storage with simple retention

```cpp
// Memory-first storage with disk persistence
#include <fstream>
#include <deque>
#include <filesystem>
#include <thread>
#include <future>

class HybridStorage {
private:
    std::deque<JsonEvent> memory_buffer_;
    mutable std::mutex memory_mutex_;
    std::unique_ptr<DiskStorage> disk_storage_;
    size_t max_memory_events_ = 10000;
    
public:
    HybridStorage(const std::string& base_path, size_t max_file_size) 
        : disk_storage_(std::make_unique<DiskStorage>(base_path, max_file_size)) {}
    
    void store_event(const JsonEvent& event) {
        // Always store in memory first
        {
            std::lock_guard<std::mutex> lock(memory_mutex_);
            memory_buffer_.push_back(event);
            
            // Simple eviction policy
            if (memory_buffer_.size() > max_memory_events_) {
                memory_buffer_.pop_front();
            }
        }
        
        // Async write to disk for persistence
        std::thread([this, event]() {
            disk_storage_->append_event(event);
        }).detach();
    }
    
    std::vector<JsonEvent> get_events_since(int64_t timestamp) const {
        std::vector<JsonEvent> result;
        
        // Check memory buffer first for recent events
        {
            std::lock_guard<std::mutex> lock(memory_mutex_);
            for (const auto& event : memory_buffer_) {
                if (event.timestamp >= timestamp) {
                    result.push_back(event);
                }
            }
        }
        
        // If we need older events, check disk storage
        if (result.empty() || (!result.empty() && result.front().timestamp > timestamp)) {
            auto disk_events = disk_storage_->read_events_since(timestamp);
            result.insert(result.begin(), disk_events.begin(), disk_events.end());
        }
        
        return result;
    }
    
    size_t get_memory_event_count() const {
        std::lock_guard<std::mutex> lock(memory_mutex_);
        return memory_buffer_.size();
    }
};

// Simple disk storage using append-only logs
class DiskStorage {
private:
    std::ofstream current_file_;
    std::string base_path_;
    size_t max_file_size_;
    mutable std::mutex file_mutex_;
    
public:
    DiskStorage(const std::string& base_path, size_t max_file_size)
        : base_path_(base_path), max_file_size_(max_file_size) {
        
        std::filesystem::create_directories(base_path);
        open_new_file();
    }
    
    ~DiskStorage() {
        std::lock_guard<std::mutex> lock(file_mutex_);
        if (current_file_.is_open()) {
            current_file_.close();
        }
    }
    
    void append_event(const JsonEvent& event) {
        std::lock_guard<std::mutex> lock(file_mutex_);
        
        nlohmann::json serialized = {
            {"id", event.id},
            {"timestamp", event.timestamp},
            {"data", event.data},
            {"metadata", {
                {"source", event.metadata.source},
                {"pipeline", event.metadata.pipeline},
                {"retry_count", event.metadata.retry_count}
            }}
        };
        
        if (event.metadata.routing_key.has_value()) {
            serialized["metadata"]["routing_key"] = event.metadata.routing_key.value();
        }
        
        current_file_ << serialized.dump() << std::endl;
        current_file_.flush();
        
        // Simple file rotation based on size
        if (current_file_.tellp() > static_cast<std::streampos>(max_file_size_)) {
            rotate_file();
        }
    }
    
    std::vector<JsonEvent> read_events_since(int64_t timestamp) const {
        std::vector<JsonEvent> events;
        
        // Simple implementation: scan all files in directory
        for (const auto& entry : std::filesystem::directory_iterator(base_path_)) {
            if (entry.path().extension() == ".log") {
                auto file_events = read_events_from_file(entry.path(), timestamp);
                events.insert(events.end(), file_events.begin(), file_events.end());
            }
        }
        
        // Sort by timestamp
        std::sort(events.begin(), events.end(), 
                  [](const JsonEvent& a, const JsonEvent& b) {
                      return a.timestamp < b.timestamp;
                  });
        
        return events;
    }
    
private:
    void open_new_file() {
        auto timestamp = std::chrono::duration_cast<std::chrono::seconds>(
            std::chrono::system_clock::now().time_since_epoch()).count();
        std::string filename = base_path_ + "/events_" + std::to_string(timestamp) + ".log";
        current_file_.open(filename, std::ios::app);
    }
    
    void rotate_file() {
        current_file_.close();
        open_new_file();
    }
    
    std::vector<JsonEvent> read_events_from_file(const std::filesystem::path& file_path, int64_t since_timestamp) const {
        std::vector<JsonEvent> events;
        std::ifstream file(file_path);
        std::string line;
        
        while (std::getline(file, line)) {
            try {
                auto json_event = nlohmann::json::parse(line);
                
                if (json_event["timestamp"].get<int64_t>() >= since_timestamp) {
                    JsonEvent event;
                    event.id = json_event["id"];
                    event.timestamp = json_event["timestamp"];
                    event.data = json_event["data"];
                    event.metadata.source = json_event["metadata"]["source"];
                    event.metadata.pipeline = json_event["metadata"]["pipeline"];
                    event.metadata.retry_count = json_event["metadata"]["retry_count"];
                    
                    if (json_event["metadata"].contains("routing_key")) {
                        event.metadata.routing_key = json_event["metadata"]["routing_key"];
                    }
                    
                    events.push_back(event);
                }
            } catch (const std::exception&) {
                // Skip malformed lines
                continue;
            }
        }
        
        return events;
    }
};
```

#### Computo Integration
**REQ-031 to REQ-040**: Leverage existing Computo library

```cpp
// Wrapper for Computo execution in streaming context
#include <computo.hpp>
#include <unordered_map>
#include <shared_mutex>

class ComputoExecutor {
private:
    std::unordered_map<std::string, std::string> compiled_scripts_;
    mutable std::shared_mutex scripts_mutex_;
    std::unique_ptr<ContextFactory> context_factory_;
    
public:
    ComputoExecutor() : context_factory_(std::make_unique<ContextFactory>()) {}
    
    JsonEvent execute_transformation(const std::string& script_id, const JsonEvent& event) {
        // Get script from cache
        std::shared_lock<std::shared_mutex> lock(scripts_mutex_);
        auto it = compiled_scripts_.find(script_id);
        if (it == compiled_scripts_.end()) {
            throw std::runtime_error("Script not found: " + script_id);
        }
        
        const std::string& script = it->second;
        lock.unlock();
        
        // Create execution context with event data and metadata
        auto context_inputs = context_factory_->create_context(event);
        
        // Execute using Computo library directly
        nlohmann::json script_json = nlohmann::json::parse(script);
        nlohmann::json result = computo::execute(script_json, context_inputs);
        
        // Create new event with transformed data
        JsonEvent transformed_event;
        transformed_event.id = generate_event_id();
        transformed_event.timestamp = event.timestamp;
        transformed_event.data = std::move(result);
        transformed_event.metadata = event.metadata;
        
        return transformed_event;
    }
    
    void add_script(const std::string& script_id, const std::string& script) {
        std::unique_lock<std::shared_mutex> lock(scripts_mutex_);
        
        // Validate script by parsing it
        try {
            nlohmann::json::parse(script);
        } catch (const nlohmann::json::exception& e) {
            throw std::runtime_error("Invalid script JSON: " + std::string(e.what()));
        }
        
        compiled_scripts_[script_id] = script;
    }
    
    void remove_script(const std::string& script_id) {
        std::unique_lock<std::shared_mutex> lock(scripts_mutex_);
        compiled_scripts_.erase(script_id);
    }
    
    std::vector<std::string> list_scripts() const {
        std::shared_lock<std::shared_mutex> lock(scripts_mutex_);
        std::vector<std::string> script_ids;
        script_ids.reserve(compiled_scripts_.size());
        
        for (const auto& pair : compiled_scripts_) {
            script_ids.push_back(pair.first);
        }
        
        return script_ids;
    }
    
private:
    std::string generate_event_id() {
        static thread_local std::random_device rd;
        static thread_local std::mt19937 gen(rd());
        static thread_local std::uniform_int_distribution<> dis(0, 15);
        
        std::string uuid;
        uuid.reserve(32);
        for (int i = 0; i < 32; ++i) {
            uuid += "0123456789abcdef"[dis(gen)];
        }
        return uuid;
    }
};

// Context factory injects stream metadata for Computo scripts
class ContextFactory {
public:
    std::vector<nlohmann::json> create_context(const JsonEvent& event) {
        std::vector<nlohmann::json> inputs;
        
        // Primary input is the event data
        inputs.push_back(event.data);
        
        // Inject metadata as additional input for Computo scripts
        nlohmann::json metadata_json = {
            {"source", event.metadata.source},
            {"pipeline", event.metadata.pipeline},
            {"retry_count", event.metadata.retry_count},
            {"event_id", event.id},
            {"timestamp", event.timestamp}
        };
        
        if (event.metadata.routing_key.has_value()) {
            metadata_json["routing_key"] = event.metadata.routing_key.value();
        }
        
        inputs.push_back(metadata_json);
        
        return inputs;
    }
};
```

### Phase 1 Testing Strategy

#### Unit Tests for Each Component
```cpp
#include <gtest/gtest.h>
#include <nlohmann/json.hpp>
#include <filesystem>

class ColloquiumTest : public ::testing::Test {
protected:
    void SetUp() override {
        test_storage_path_ = "/tmp/test_storage_" + std::to_string(std::random_device{}());
        storage_ = std::make_unique<HybridStorage>(test_storage_path_, 1024 * 1024);
        executor_ = std::make_unique<ComputoExecutor>();
    }
    
    void TearDown() override {
        // Clean up test storage
        std::filesystem::remove_all(test_storage_path_);
    }
    
    JsonEvent create_test_event(const std::string& id, const nlohmann::json& data) {
        JsonEvent event;
        event.id = id;
        event.timestamp = JsonEvent::current_timestamp();
        event.data = data;
        event.metadata = EventMetadata{"test", "test-pipeline", std::nullopt, 0};
        return event;
    }
    
    std::string test_storage_path_;
    std::unique_ptr<HybridStorage> storage_;
    std::unique_ptr<ComputoExecutor> executor_;
};

TEST_F(ColloquiumTest, EventStorageAndRetrieval) {
    auto event = create_test_event("test-event-001", nlohmann::json{{"test", "data"}});
    
    storage_->store_event(event);
    
    // Give async storage time to complete
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    
    auto retrieved = storage_->get_events_since(event.timestamp - 1);
    ASSERT_EQ(retrieved.size(), 1);
    EXPECT_EQ(retrieved[0].data, event.data);
    EXPECT_EQ(retrieved[0].id, event.id);
}

TEST_F(ColloquiumTest, ComputoTransformation) {
    // Simple transformation script that adds 10 to the value
    std::string script = R"(["obj", ["result", ["+", ["get", ["$input"], "/value"], 10]]])";
    executor_->add_script("test", script);
    
    auto event = create_test_event("test-event-002", nlohmann::json{{"value", 5}});
    
    auto result = executor_->execute_transformation("test", event);
    EXPECT_EQ(result.data["result"], 15);
    EXPECT_EQ(result.metadata.source, event.metadata.source);
}

TEST_F(ColloquiumTest, ComputoContextInjection) {
    // Script that accesses metadata
    std::string script = R"(["obj", ["event_id", ["get", ["$inputs"], "/1/event_id"]], ["data", ["$input"]]])";
    executor_->add_script("metadata_test", script);
    
    auto event = create_test_event("test-event-003", nlohmann::json{{"value", 42}});
    
    auto result = executor_->execute_transformation("metadata_test", event);
    EXPECT_EQ(result.data["event_id"], event.id);
    EXPECT_EQ(result.data["data"], event.data);
}

TEST_F(ColloquiumTest, ErrorHandling) {
    // Test with invalid script JSON
    EXPECT_THROW(executor_->add_script("invalid", "invalid json"), std::runtime_error);
    
    // Test with non-existent script
    auto event = create_test_event("test-event-004", nlohmann::json{{"value", 5}});
    EXPECT_THROW(executor_->execute_transformation("nonexistent", event), std::runtime_error);
}

TEST_F(ColloquiumTest, MultipleEvents) {
    std::vector<JsonEvent> events;
    for (int i = 0; i < 100; ++i) {
        auto event = create_test_event("event-" + std::to_string(i), 
                                       nlohmann::json{{"index", i}});
        events.push_back(event);
        storage_->store_event(event);
    }
    
    // Wait for async storage
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    
    auto retrieved = storage_->get_events_since(events[0].timestamp);
    EXPECT_EQ(retrieved.size(), 100);
    
    // Events should be sorted by timestamp
    for (size_t i = 1; i < retrieved.size(); ++i) {
        EXPECT_GE(retrieved[i].timestamp, retrieved[i-1].timestamp);
    }
}
```

#### Integration Tests
```cpp
TEST_F(ColloquiumTest, EndToEndPipeline) {
    // Start HTTP source
    HttpSource source("127.0.0.1", 8080, 1024 * 1024);
    std::thread source_thread([&source]() {
        source.start();
    });
    
    // Wait for startup
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    
    // Send test event via HTTP
    httplib::Client client("127.0.0.1", 8080);
    nlohmann::json test_data = {{"message", "hello world"}};
    
    auto response = client.Post("/events", test_data.dump(), "application/json");
    ASSERT_TRUE(response);
    EXPECT_EQ(response->status, 202);
    
    // Get event from source
    auto event = source.get_next_event();
    ASSERT_TRUE(event.has_value());
    EXPECT_EQ(event->data["message"], "hello world");
    
    // Clean up
    source.stop();
    source_thread.join();
}

TEST_F(ColloquiumTest, PerformanceBaseline) {
    const int num_events = 1000;
    auto start_time = std::chrono::high_resolution_clock::now();
    
    // Add transformation script
    std::string script = R"(["obj", ["doubled", ["*", ["get", ["$input"], "/value"], 2]]])";
    executor_->add_script("double", script);
    
    // Process events
    for (int i = 0; i < num_events; ++i) {
        auto event = create_test_event("perf-" + std::to_string(i), 
                                       nlohmann::json{{"value", i}});
        
        auto result = executor_->execute_transformation("double", event);
        EXPECT_EQ(result.data["doubled"], i * 2);
    }
    
    auto end_time = std::chrono::high_resolution_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end_time - start_time);
    
    // Should process at least 100 events per second
    double events_per_second = (num_events * 1000.0) / duration.count();
    EXPECT_GT(events_per_second, 100.0) << "Performance too slow: " << events_per_second << " events/sec";
}
```

### Phase 1 Performance Targets

#### Throughput Requirements
- **Target**: 1,000 events/second sustained throughput
- **Measurement**: Load test with continuous JSON POST requests
- **Acceptance**: 99th percentile latency < 50ms under target load

#### Memory Requirements  
- **Target**: < 512MB memory usage for basic workload
- **Measurement**: Monitor RSS during sustained load test
- **Acceptance**: No memory leaks, stable memory usage over 24 hours

#### CPU Requirements
- **Target**: < 50% CPU usage on single core under target load
- **Measurement**: Monitor CPU usage during load testing
- **Acceptance**: Linear scaling relationship between load and CPU usage

### Phase 1 Configuration System

```cpp
// Simple configuration management using nlohmann::json
#include <fstream>

struct ColloquiumConfig {
    struct ServerConfig {
        std::string host = "0.0.0.0";
        int port = 8080;
        size_t max_body_size = 1024 * 1024; // 1MB
    } server;
    
    struct StorageConfig {
        std::string disk_path = "./data";
        size_t max_file_size = 100 * 1024 * 1024; // 100MB
        size_t memory_buffer_size = 10000;
        int retention_days = 7;
    } storage;
    
    struct ProcessingConfig {
        int worker_threads = 4;
        size_t max_queue_size = 1000;
    } processing;
    
    std::unordered_map<std::string, std::string> scripts;
    
    static ColloquiumConfig load_from_file(const std::string& config_path) {
        std::ifstream file(config_path);
        if (!file.is_open()) {
            throw std::runtime_error("Cannot open config file: " + config_path);
        }
        
        nlohmann::json config_json;
        file >> config_json;
        
        ColloquiumConfig config;
        
        if (config_json.contains("server")) {
            auto& server = config_json["server"];
            config.server.host = server.value("host", config.server.host);
            config.server.port = server.value("port", config.server.port);
            config.server.max_body_size = server.value("max_body_size", config.server.max_body_size);
        }
        
        if (config_json.contains("storage")) {
            auto& storage = config_json["storage"];
            config.storage.disk_path = storage.value("disk_path", config.storage.disk_path);
            config.storage.max_file_size = storage.value("max_file_size", config.storage.max_file_size);
            config.storage.memory_buffer_size = storage.value("memory_buffer_size", config.storage.memory_buffer_size);
            config.storage.retention_days = storage.value("retention_days", config.storage.retention_days);
        }
        
        if (config_json.contains("processing")) {
            auto& processing = config_json["processing"];
            config.processing.worker_threads = processing.value("worker_threads", config.processing.worker_threads);
            config.processing.max_queue_size = processing.value("max_queue_size", config.processing.max_queue_size);
        }
        
        if (config_json.contains("scripts")) {
            for (auto& [key, value] : config_json["scripts"].items()) {
                config.scripts[key] = value.dump();
            }
        }
        
        return config;
    }
};
```

Example configuration file:
```json
{
  "server": {
    "host": "0.0.0.0",
    "port": 8080,
    "max_body_size": 1048576
  },
  "storage": {
    "disk_path": "./data",
    "max_file_size": 104857600,
    "memory_buffer_size": 10000,
    "retention_days": 7
  },
  "processing": {
    "worker_threads": 4,
    "max_queue_size": 1000
  },
  "scripts": {
    "identity": ["$input"],
    "add_timestamp": ["obj", ["data", ["$input"]], ["processed_at", ["get", ["$inputs"], "/1/timestamp"]]]
  }
}
```

### Phase 1 Monitoring

#### Essential Metrics
```cpp
// Simple metrics collection
class MetricsCollector {
private:
    std::atomic<uint64_t> events_processed_{0};
    std::atomic<uint64_t> events_failed_{0};
    std::atomic<uint64_t> total_processing_time_ms_{0};
    std::atomic<uint32_t> active_connections_{0};
    
public:
    void record_event_processed(uint64_t processing_time_ms) {
        events_processed_.fetch_add(1);
        total_processing_time_ms_.fetch_add(processing_time_ms);
    }
    
    void record_event_failed() {
        events_failed_.fetch_add(1);
    }
    
    void record_connection_opened() {
        active_connections_.fetch_add(1);
    }
    
    void record_connection_closed() {
        active_connections_.fetch_sub(1);
    }
    
    nlohmann::json get_metrics() const {
        uint64_t processed = events_processed_.load();
        uint64_t failed = events_failed_.load();
        uint64_t total_time = total_processing_time_ms_.load();
        
        return nlohmann::json{
            {"events_processed", processed},
            {"events_failed", failed},
            {"active_connections", active_connections_.load()},
            {"average_processing_time_ms", processed > 0 ? total_time / processed : 0},
            {"error_rate", processed > 0 ? (double)failed / (processed + failed) : 0.0}
        };
    }
};
```

#### Health Checks
```cpp
// Simple health check endpoint
class HealthChecker {
private:
    StreamEngine* engine_;
    HybridStorage* storage_;
    
public:
    HealthChecker(StreamEngine* engine, HybridStorage* storage)
        : engine_(engine), storage_(storage) {}
    
    nlohmann::json check_health() const {
        nlohmann::json health = {
            {"status", "healthy"},
            {"timestamp", JsonEvent::current_timestamp()},
            {"components", nlohmann::json::array()}
        };
        
        // Check engine status
        health["components"].push_back({
            {"name", "stream_engine"},
            {"status", engine_->is_running() ? "healthy" : "stopped"}
        });
        
        // Check storage status
        try {
            size_t event_count = storage_->get_memory_event_count();
            health["components"].push_back({
                {"name", "storage"},
                {"status", "healthy"},
                {"memory_events", event_count}
            });
        } catch (const std::exception& e) {
            health["components"].push_back({
                {"name", "storage"},
                {"status", "unhealthy"},
                {"error", e.what()}
            });
            health["status"] = "degraded";
        }
        
        return health;
    }
};
```

## Phase 2: AI Provider Integration (Months 4-6)

### AI Provider Architecture

```cpp
// Generic AI provider interface
#include <curl/curl.h>
#include <future>

class AIProvider {
public:
    virtual ~AIProvider() = default;
    virtual std::future<nlohmann::json> send_request(const nlohmann::json& request) = 0;
    virtual std::string get_provider_name() const = 0;
    virtual double estimate_cost(const nlohmann::json& request) const = 0;
    virtual bool is_healthy() const = 0;
};

// OpenAI implementation
class OpenAIProvider : public AIProvider {
private:
    std::string api_key_;
    std::string base_url_;
    std::unique_ptr<RateLimiter> rate_limiter_;
    std::unique_ptr<CostTracker> cost_tracker_;
    
    // CURL wrapper for HTTP requests
    struct CurlResponse {
        std::string data;
        long status_code;
        std::string error;
    };
    
    static size_t WriteCallback(void* contents, size_t size, size_t nmemb, CurlResponse* response) {
        size_t total_size = size * nmemb;
        response->data.append(static_cast<char*>(contents), total_size);
        return total_size;
    }
    
public:
    OpenAIProvider(const std::string& api_key, const std::string& base_url = "https://api.openai.com/v1")
        : api_key_(api_key), base_url_(base_url) {
        
        rate_limiter_ = std::make_unique<TokenBucketLimiter>(1000, 150000); // 1000 req/min, 150K tokens/min
        cost_tracker_ = std::make_unique<CostTracker>();
    }
    
    std::future<nlohmann::json> send_request(const nlohmann::json& request) override {
        return std::async(std::launch::async, [this, request]() -> nlohmann::json {
            // Estimate tokens and wait for rate limit
            uint32_t estimated_tokens = estimate_tokens(request);
            rate_limiter_->acquire_tokens(estimated_tokens);
            
            // Transform to OpenAI format
            nlohmann::json openai_request = transform_to_openai_format(request);
            
            // Send HTTP request
            auto response = send_http_request("/chat/completions", openai_request);
            
            if (response.status_code != 200) {
                throw std::runtime_error("OpenAI API error: " + std::to_string(response.status_code) + " " + response.data);
            }
            
            nlohmann::json response_json = nlohmann::json::parse(response.data);
            
            // Track costs
            cost_tracker_->record_usage(response_json);
            
            return transform_from_openai_format(response_json);
        });
    }
    
    std::string get_provider_name() const override {
        return "openai";
    }
    
    double estimate_cost(const nlohmann::json& request) const override {
        uint32_t tokens = estimate_tokens(request);
        return tokens * 0.00003; // Approximate cost per token
    }
    
    bool is_healthy() const override {
        return true; // TODO: Implement actual health check
    }
    
private:
    CurlResponse send_http_request(const std::string& endpoint, const nlohmann::json& data) {
        CURL* curl = curl_easy_init();
        CurlResponse response;
        
        if (curl) {
            std::string url = base_url_ + endpoint;
            std::string json_data = data.dump();
            
            struct curl_slist* headers = nullptr;
            std::string auth_header = "Authorization: Bearer " + api_key_;
            headers = curl_slist_append(headers, "Content-Type: application/json");
            headers = curl_slist_append(headers, auth_header.c_str());
            
            curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
            curl_easy_setopt(curl, CURLOPT_POSTFIELDS, json_data.c_str());
            curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);
            curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
            curl_easy_setopt(curl, CURLOPT_WRITEDATA, &response);
            curl_easy_setopt(curl, CURLOPT_TIMEOUT, 30L);
            
            CURLcode res = curl_easy_perform(curl);
            if (res != CURLE_OK) {
                response.error = curl_easy_strerror(res);
                response.status_code = 0;
            } else {
                curl_easy_getinfo(curl, CURLINFO_RESPONSE_CODE, &response.status_code);
            }
            
            curl_slist_free_all(headers);
            curl_easy_cleanup(curl);
        }
        
        return response;
    }
    
    nlohmann::json transform_to_openai_format(const nlohmann::json& request) {
        // Simple transformation from universal format to OpenAI format
        return nlohmann::json{
            {"model", request.value("model", "gpt-3.5-turbo")},
            {"messages", request.value("messages", nlohmann::json::array())},
            {"max_tokens", request.value("max_tokens", 1000)},
            {"temperature", request.value("temperature", 0.7)}
        };
    }
    
    nlohmann::json transform_from_openai_format(const nlohmann::json& response) {
        // Transform OpenAI response to universal format
        return nlohmann::json{
            {"provider", "openai"},
            {"response", response["choices"][0]["message"]["content"]},
            {"usage", response["usage"]},
            {"model", response["model"]}
        };
    }
    
    uint32_t estimate_tokens(const nlohmann::json& request) const {
        // Simple token estimation (4 chars = 1 token approximation)
        std::string content = request.dump();
        return static_cast<uint32_t>(content.length() / 4);
    }
};
```

### Rate Limiting Implementation

```cpp
// Token bucket rate limiter with burst support
class TokenBucketLimiter {
private:
    mutable std::mutex mutex_;
    double tokens_;
    double max_tokens_;
    double refill_rate_; // tokens per second
    std::chrono::steady_clock::time_point last_refill_;
    
public:
    TokenBucketLimiter(double max_tokens, double refill_rate)
        : tokens_(max_tokens), max_tokens_(max_tokens), refill_rate_(refill_rate),
          last_refill_(std::chrono::steady_clock::now()) {}
    
    bool try_acquire_tokens(uint32_t tokens_needed) {
        std::lock_guard<std::mutex> lock(mutex_);
        refill_tokens();
        
        if (tokens_ >= tokens_needed) {
            tokens_ -= tokens_needed;
            return true;
        }
        
        return false;
    }
    
    void acquire_tokens(uint32_t tokens_needed) {
        while (!try_acquire_tokens(tokens_needed)) {
            // Simple exponential backoff
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
    
    double get_available_tokens() const {
        std::lock_guard<std::mutex> lock(mutex_);
        const_cast<TokenBucketLimiter*>(this)->refill_tokens();
        return tokens_;
    }
    
private:
    void refill_tokens() {
        auto now = std::chrono::steady_clock::now();
        auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(now - last_refill_).count();
        
        if (elapsed > 0) {
            double tokens_to_add = (elapsed / 1000.0) * refill_rate_;
            tokens_ = std::min(max_tokens_, tokens_ + tokens_to_add);
            last_refill_ = now;
        }
    }
};
```

### Cost Tracking and Management

```cpp
// Real-time cost tracking with budget enforcement
class CostTracker {
private:
    mutable std::mutex mutex_;
    double current_cost_ = 0.0;
    double daily_budget_;
    std::deque<CostEntry> cost_history_;
    
    struct CostEntry {
        std::chrono::system_clock::time_point timestamp;
        double cost;
        std::string provider;
        std::string model;
    };
    
public:
    CostTracker(double daily_budget = 100.0) : daily_budget_(daily_budget) {}
    
    void record_usage(const nlohmann::json& response) {
        std::lock_guard<std::mutex> lock(mutex_);
        
        double cost = calculate_cost(response);
        current_cost_ += cost;
        
        cost_history_.push_back({
            std::chrono::system_clock::now(),
            cost,
            response.value("provider", "unknown"),
            response.value("model", "unknown")
        });
        
        // Clean old entries (keep last 24 hours)
        auto cutoff = std::chrono::system_clock::now() - std::chrono::hours(24);
        while (!cost_history_.empty() && cost_history_.front().timestamp < cutoff) {
            current_cost_ -= cost_history_.front().cost;
            cost_history_.pop_front();
        }
        
        // Check budget limits
        if (current_cost_ > daily_budget_) {
            throw std::runtime_error("Daily budget exceeded: $" + std::to_string(current_cost_));
        }
    }
    
    double get_current_cost() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return current_cost_;
    }
    
    double get_remaining_budget() const {
        std::lock_guard<std::mutex> lock(mutex_);
        return std::max(0.0, daily_budget_ - current_cost_);
    }
    
    nlohmann::json get_cost_summary() const {
        std::lock_guard<std::mutex> lock(mutex_);
        
        std::unordered_map<std::string, double> by_provider;
        std::unordered_map<std::string, uint32_t> request_counts;
        
        for (const auto& entry : cost_history_) {
            by_provider[entry.provider] += entry.cost;
            request_counts[entry.provider]++;
        }
        
        nlohmann::json summary = {
            {"current_cost", current_cost_},
            {"daily_budget", daily_budget_},
            {"remaining_budget", daily_budget_ - current_cost_},
            {"by_provider", by_provider},
            {"request_counts", request_counts}
        };
        
        return summary;
    }
    
private:
    double calculate_cost(const nlohmann::json& response) {
        // Simple cost calculation based on token usage
        if (response.contains("usage")) {
            auto usage = response["usage"];
            uint32_t input_tokens = usage.value("prompt_tokens", 0);
            uint32_t output_tokens = usage.value("completion_tokens", 0);
            
            // Approximate costs (should be configurable per provider/model)
            return (input_tokens * 0.00003) + (output_tokens * 0.00006);
        }
        
        return 0.01; // Default cost if usage not available
    }
};
```

### AI-Aware Stream Processing

```cpp
// Enhanced pipeline with AI provider integration
class AIStreamPipeline {
private:
    std::unique_ptr<ProcessingPipeline> base_pipeline_;
    std::unique_ptr<ProviderRegistry> provider_registry_;
    std::unique_ptr<RequestRouter> request_router_;
    
public:
    AIStreamPipeline() {
        base_pipeline_ = std::make_unique<ProcessingPipeline>();
        provider_registry_ = std::make_unique<ProviderRegistry>();
        request_router_ = std::make_unique<RequestRouter>(provider_registry_.get());
    }
    
    std::future<JsonEvent> process_ai_request(const JsonEvent& event) {
        return std::async(std::launch::async, [this, event]() -> JsonEvent {
            // Extract AI request from event
            nlohmann::json ai_request = extract_ai_request(event);
            
            // Select optimal provider based on cost and availability
            std::string provider_id = request_router_->select_provider(ai_request);
            
            // Get provider and send request
            auto provider = provider_registry_->get_provider(provider_id);
            if (!provider) {
                throw std::runtime_error("Provider not available: " + provider_id);
            }
            
            auto response_future = provider->send_request(ai_request);
            nlohmann::json ai_response = response_future.get();
            
            // Create result event with AI response
            JsonEvent result_event;
            result_event.id = generate_event_id();
            result_event.timestamp = JsonEvent::current_timestamp();
            result_event.data = ai_response;
            result_event.metadata = event.metadata;
            result_event.metadata.routing_key = provider_id;
            
            return result_event;
        });
    }
    
    void add_provider(const std::string& provider_id, std::unique_ptr<AIProvider> provider) {
        provider_registry_->add_provider(provider_id, std::move(provider));
    }
    
private:
    nlohmann::json extract_ai_request(const JsonEvent& event) {
        // Simple extraction - assume event data contains AI request
        return event.data;
    }
    
    std::string generate_event_id() {
        static thread_local std::random_device rd;
        static thread_local std::mt19937 gen(rd());
        static thread_local std::uniform_int_distribution<> dis(0, 15);
        
        std::string uuid;
        uuid.reserve(32);
        for (int i = 0; i < 32; ++i) {
            uuid += "0123456789abcdef"[dis(gen)];
        }
        return uuid;
    }
};

// Provider registry for managing multiple AI providers
class ProviderRegistry {
private:
    std::unordered_map<std::string, std::unique_ptr<AIProvider>> providers_;
    mutable std::shared_mutex providers_mutex_;
    
public:
    void add_provider(const std::string& provider_id, std::unique_ptr<AIProvider> provider) {
        std::unique_lock<std::shared_mutex> lock(providers_mutex_);
        providers_[provider_id] = std::move(provider);
    }
    
    AIProvider* get_provider(const std::string& provider_id) {
        std::shared_lock<std::shared_mutex> lock(providers_mutex_);
        auto it = providers_.find(provider_id);
        return (it != providers_.end()) ? it->second.get() : nullptr;
    }
    
    std::vector<std::string> get_available_providers() const {
        std::shared_lock<std::shared_mutex> lock(providers_mutex_);
        std::vector<std::string> provider_ids;
        provider_ids.reserve(providers_.size());
        
        for (const auto& pair : providers_) {
            if (pair.second->is_healthy()) {
                provider_ids.push_back(pair.first);
            }
        }
        
        return provider_ids;
    }
};

// Simple request router for provider selection
class RequestRouter {
private:
    ProviderRegistry* provider_registry_;
    
public:
    RequestRouter(ProviderRegistry* registry) : provider_registry_(registry) {}
    
    std::string select_provider(const nlohmann::json& request) {
        auto available_providers = provider_registry_->get_available_providers();
        
        if (available_providers.empty()) {
            throw std::runtime_error("No providers available");
        }
        
        // Simple cost-based selection
        std::string best_provider = available_providers[0];
        double best_cost = std::numeric_limits<double>::max();
        
        for (const auto& provider_id : available_providers) {
            auto provider = provider_registry_->get_provider(provider_id);
            if (provider) {
                double cost = provider->estimate_cost(request);
                if (cost < best_cost) {
                    best_cost = cost;
                    best_provider = provider_id;
                }
            }
        }
        
        return best_provider;
    }
};
```

### Provider Health Monitoring

```cpp
// Circuit breaker pattern for provider health
class ProviderHealthMonitor {
private:
    enum class CircuitState {
        Closed,    // Normal operation
        Open,      // Provider unavailable, requests fail fast
        HalfOpen   // Testing if provider has recovered
    };
    
    struct CircuitBreakerInfo {
        CircuitState state = CircuitState::Closed;
        uint32_t failure_count = 0;
        std::chrono::steady_clock::time_point last_failure_time;
        std::chrono::steady_clock::time_point state_change_time;
    };
    
    std::unordered_map<std::string, CircuitBreakerInfo> circuit_breakers_;
    mutable std::mutex mutex_;
    
    const uint32_t failure_threshold_ = 5;
    const std::chrono::seconds timeout_{30};
    const std::chrono::seconds half_open_timeout_{10};
    
public:
    bool is_provider_available(const std::string& provider_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto& breaker = circuit_breakers_[provider_id];
        
        auto now = std::chrono::steady_clock::now();
        
        switch (breaker.state) {
            case CircuitState::Closed:
                return true;
                
            case CircuitState::Open:
                if (now - breaker.state_change_time > timeout_) {
                    breaker.state = CircuitState::HalfOpen;
                    breaker.state_change_time = now;
                    return true;
                }
                return false;
                
            case CircuitState::HalfOpen:
                if (now - breaker.state_change_time > half_open_timeout_) {
                    breaker.state = CircuitState::Open;
                    breaker.state_change_time = now;
                    return false;
                }
                return true;
        }
        
        return false;
    }
    
    void record_success(const std::string& provider_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto& breaker = circuit_breakers_[provider_id];
        
        breaker.failure_count = 0;
        breaker.state = CircuitState::Closed;
        breaker.state_change_time = std::chrono::steady_clock::now();
    }
    
    void record_failure(const std::string& provider_id) {
        std::lock_guard<std::mutex> lock(mutex_);
        auto& breaker = circuit_breakers_[provider_id];
        
        breaker.failure_count++;
        breaker.last_failure_time = std::chrono::steady_clock::now();
        
        if (breaker.failure_count >= failure_threshold_) {
            breaker.state = CircuitState::Open;
            breaker.state_change_time = breaker.last_failure_time;
        }
    }
    
    nlohmann::json get_health_status() const {
        std::lock_guard<std::mutex> lock(mutex_);
        nlohmann::json status = nlohmann::json::object();
        
        for (const auto& pair : circuit_breakers_) {
            std::string state_str;
            switch (pair.second.state) {
                case CircuitState::Closed: state_str = "closed"; break;
                case CircuitState::Open: state_str = "open"; break;
                case CircuitState::HalfOpen: state_str = "half_open"; break;
            }
            
            status[pair.first] = {
                {"state", state_str},
                {"failure_count", pair.second.failure_count},
                {"available", pair.second.state != CircuitState::Open}
            };
        }
        
        return status;
    }
};
```

### Phase 2 Configuration Enhancement

```json
{
  "server": {
    "host": "0.0.0.0",
    "port": 8080,
    "max_body_size": 1048576
  },
  "storage": {
    "disk_path": "./data",
    "max_file_size": 104857600,
    "memory_buffer_size": 10000,
    "retention_days": 7
  },
  "processing": {
    "worker_threads": 4,
    "max_queue_size": 1000
  },
  "ai_providers": {
    "openai": {
      "api_key": "${OPENAI_API_KEY}",
      "base_url": "https://api.openai.com/v1",
      "rate_limit": {
        "requests_per_minute": 1000,
        "tokens_per_minute": 150000
      },
      "models": ["gpt-4", "gpt-4-turbo", "gpt-3.5-turbo"],
      "cost_per_token": {
        "input": 0.00003,
        "output": 0.00006
      }
    },
    "anthropic": {
      "api_key": "${ANTHROPIC_API_KEY}",
      "base_url": "https://api.anthropic.com/v1",
      "rate_limit": {
        "requests_per_minute": 500,
        "tokens_per_minute": 100000
      },
      "models": ["claude-3-haiku", "claude-3-sonnet", "claude-3-opus"],
      "cost_per_token": {
        "input": 0.000015,
        "output": 0.000075
      }
    }
  },
  "cost_management": {
    "daily_budget": 100.0,
    "alert_threshold": 0.8,
    "auto_stop_on_budget": true
  },
  "scripts": {
    "ai_request_transform": [
      "obj",
      ["provider", ["get", ["$input"], "/provider"]],
      ["model", ["get", ["$input"], "/model"]],
      ["messages", ["get", ["$input"], "/messages"]],
      ["max_tokens", ["get", ["$input"], "/max_tokens"]]
    ]
  }
}
```

## Success Criteria and Quality Gates

### Phase 1 Success Criteria
- **Functional**: All 30 core requirements implemented and tested
- **Performance**: 1,000 events/second sustained throughput with <50ms p99 latency
- **Reliability**: 24-hour stress test with zero memory leaks or crashes
- **Documentation**: Complete API documentation and deployment guide

### Phase 2 Success Criteria
- **AI Integration**: All major providers integrated with working rate limiting
- **Cost Management**: Accurate cost tracking with budget enforcement
- **Monitoring**: Comprehensive metrics and alerting for all components
- **Security**: Authentication and authorization working in production

### Quality Gates Between Phases

#### Performance Gates
- Load testing must pass before advancing to next phase
- Memory usage must remain stable under sustained load
- CPU utilization must scale linearly with load
- No performance regressions from previous phase

#### Security Gates
- Security scan must pass with zero high-severity vulnerabilities
- All authentication and authorization tests must pass
- Encryption must be properly implemented for all data
- Security review by external auditor

#### Reliability Gates
- Chaos engineering tests must pass
- Failure recovery tests must pass
- Data integrity tests must pass under all failure scenarios
- Backup and restore procedures must be tested

This technical implementation strategy provides a practical, step-by-step approach to building Colloquium while maintaining the architectural principles and quality standards established by Permuto and Computo. Each phase builds incrementally on proven foundations, with rigorous testing and quality gates to prevent the over-engineering pitfalls common in stream processing platforms.