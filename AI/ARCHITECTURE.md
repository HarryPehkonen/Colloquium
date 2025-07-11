# ARCHITECTURE.md

## Colloquium Clean Architecture Design

Colloquium extends the proven architectural principles from Permuto and Computo to create an AI-native stream processing platform. This document defines the clean architecture patterns, component separation, and design principles that ensure scalable, maintainable, and reliable real-time data processing using modern C++ practices.

## Core Architectural Principles

### 1. Clean Separation of Concerns
Building on Computo's library/CLI separation pattern:

- **Stream Engine Core**: Thread-safe, minimal, high-performance event processing
- **Computo Integration Layer**: Isolated transformation execution with zero engine impact
- **AI Provider Layer**: Pluggable connectors with provider-specific optimizations
- **Management Layer**: Configuration, monitoring, and operational tools
- **No Cross-Layer Dependencies**: Clear interfaces between all architectural layers

### 2. JSON-Native Design
Extending Computo's JSON-first philosophy to stream processing:

- **Zero-Copy JSON Operations**: Stream processing without deserialization overhead
- **Native JSON Routing**: Content-based routing using JSON structure and values
- **JSON Schema Evolution**: Automatic handling of schema changes in event streams
- **Type-Safe Transformations**: Maintain JSON type information throughout pipeline

### 3. Thread Safety Through Pure Functions
Following Computo's proven thread safety model:

- **Immutable Event Processing**: Events are never modified in-place
- **Pure Transformation Functions**: Computo scripts remain side-effect free
- **Thread-Local Context**: Execution context isolated per processing thread
- **Lock-Free Data Structures**: High-performance concurrent collections where needed

### 4. Operational Simplicity
Maintaining the single binary deployment philosophy:

- **Self-Contained Deployment**: All dependencies bundled in single executable
- **Configuration-Driven**: YAML-based pipeline definitions with hot reloading
- **Built-in Observability**: Metrics, logging, and health checks included
- **Graceful Degradation**: Intelligent fallback behavior for partial failures

## Component Architecture

### Core Components Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Management Layer                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │ Config Mgmt │ │ Monitoring  │ │      Web UI/API         │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   AI Provider Layer                        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │ OpenAI      │ │ Anthropic   │ │     Provider            │ │
│  │ Connector   │ │ Connector   │ │     Registry            │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                Computo Integration Layer                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │ Script      │ │ Context     │ │     Performance         │ │
│  │ Executor    │ │ Manager     │ │     Optimizer           │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    Stream Engine Core                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │ Event       │ │ Processing  │ │       Storage           │ │
│  │ Router      │ │ Pipeline    │ │       Manager           │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Layer-by-Layer Architecture

#### Stream Engine Core (Highest Performance, Zero Dependencies)

**Event Router Component**
```cpp
#include <vector>
#include <memory>
#include <shared_mutex>
#include <atomic>
#include <unordered_map>
#include <nlohmann/json.hpp>

class EventRouter {
private:
    mutable std::shared_mutex routes_mutex_;
    std::vector<std::unique_ptr<Route>> routes_;
    std::shared_ptr<MetricsCollector> metrics_;
    std::atomic<uint64_t> route_version_{0};
    
public:
    struct RouteTarget {
        std::string pipeline_id;
        std::string sink_id;
        nlohmann::json metadata;
    };
    
    // Lock-free routing based on JSON content
    std::vector<RouteTarget> route_event(const JsonEvent& event) {
        std::shared_lock<std::shared_mutex> lock(routes_mutex_);
        std::vector<RouteTarget> targets;
        
        // Zero allocation for common routing patterns
        for (const auto& route : routes_) {
            if (route->matches(event)) {
                targets.emplace_back(route->get_target());
            }
        }
        
        metrics_->increment_counter("events_routed", targets.size());
        return targets;
    }
    
    // Atomic route updates with zero downtime
    void update_routes(std::vector<std::unique_ptr<Route>> new_routes) {
        {
            std::unique_lock<std::shared_mutex> lock(routes_mutex_);
            routes_ = std::move(new_routes);
            route_version_.fetch_add(1);
        }
        metrics_->increment_counter("route_updates");
    }
    
    uint64_t get_route_version() const noexcept {
        return route_version_.load();
    }
};
```

**Processing Pipeline Component**
```cpp
#include <vector>
#include <memory>
#include <functional>
#include <future>

class ProcessingPipeline {
private:
    std::vector<std::unique_ptr<ProcessingStage>> stages_;
    std::shared_ptr<ErrorHandler> error_handler_;
    std::shared_ptr<MetricsCollector> metrics_;
    
public:
    // Immutable event processing through stages
    std::future<JsonEvent> process_event(JsonEvent event) {
        return std::async(std::launch::async, [this, event = std::move(event)]() mutable {
            auto start_time = std::chrono::steady_clock::now();
            
            try {
                // Each stage returns new event, never modifies input
                for (const auto& stage : stages_) {
                    event = stage->process(std::move(event));
                }
                
                auto duration = std::chrono::steady_clock::now() - start_time;
                metrics_->record_histogram("pipeline_duration_ms", 
                    std::chrono::duration_cast<std::chrono::milliseconds>(duration).count());
                
                return event;
            } catch (const ProcessingError& e) {
                // Automatic error handling and recovery
                return error_handler_->handle_error(std::move(event), e);
            }
        });
    }
    
    void add_stage(std::unique_ptr<ProcessingStage> stage) {
        stages_.emplace_back(std::move(stage));
    }
    
    size_t stage_count() const noexcept {
        return stages_.size();
    }
};
```

**Storage Manager Component**
```cpp
#include <unordered_map>
#include <memory>
#include <string>
#include <chrono>

class StorageManager {
private:
    std::unordered_map<StorageType, std::unique_ptr<StorageBackend>> backends_;
    RetentionPolicy retention_policy_;
    std::unique_ptr<CompactionScheduler> compaction_scheduler_;
    
public:
    // Persistent event storage with configurable backends
    std::future<EventId> store_event(const JsonEvent& event) {
        return std::async(std::launch::async, [this, &event]() {
            auto backend = backends_[StorageType::PRIMARY].get();
            auto event_id = backend->store(event);
            
            // Automatic compression and retention management
            if (retention_policy_.should_compress(event)) {
                compaction_scheduler_->schedule_compression(event_id);
            }
            
            return event_id;
        });
    }
    
    // Efficient event replay for processing recovery
    std::unique_ptr<EventStream> replay_events(
        std::chrono::system_clock::time_point from,
        std::chrono::system_clock::time_point to) {
        
        auto backend = backends_[StorageType::PRIMARY].get();
        return backend->create_stream(from, to);
    }
    
    void add_backend(StorageType type, std::unique_ptr<StorageBackend> backend) {
        backends_[type] = std::move(backend);
    }
};
```

#### Computo Integration Layer (Performance Optimized)

**Script Executor Component**
```cpp
#include <unordered_map>
#include <shared_mutex>
#include <thread>
#include <future>
#include "computo.hpp"

class ComputoScriptExecutor {
private:
    mutable std::shared_mutex scripts_mutex_;
    std::unordered_map<ScriptId, CompiledScript> compiled_scripts_;
    std::unique_ptr<ThreadPool> execution_pool_;
    ContextFactory context_factory_;
    
public:
    // Pre-compiled script execution for maximum performance
    std::future<JsonEvent> execute_script(
        const ScriptId& script_id,
        JsonEvent event,
        const ExecutionContext& base_context) {
        
        return execution_pool_->submit([this, script_id, event = std::move(event), base_context]() mutable {
            std::shared_lock<std::shared_mutex> lock(scripts_mutex_);
            
            auto script_it = compiled_scripts_.find(script_id);
            if (script_it == compiled_scripts_.end()) {
                throw ExecutionError("Script not found: " + script_id);
            }
            
            // Thread-safe context isolation
            auto context = context_factory_.create_context(base_context, event);
            
            // Execute with automatic resource management
            auto result = computo::execute(script_it->second.bytecode, event.data, context);
            
            // Create new event with transformed data
            JsonEvent output_event = event;
            output_event.data = result;
            output_event.metadata.add_transformation(script_id);
            
            return output_event;
        });
    }
    
    // Computo script compilation with optimization
    CompilationResult compile_script(const std::string& script_text) {
        try {
            auto compiled = computo::compile(script_text);
            
            {
                std::unique_lock<std::shared_mutex> lock(scripts_mutex_);
                auto script_id = generate_script_id();
                compiled_scripts_[script_id] = std::move(compiled);
                return {script_id, true, ""};
            }
        } catch (const computo::ComputoException& e) {
            return {"", false, e.what()};
        }
    }
    
private:
    ScriptId generate_script_id() {
        return "script_" + std::to_string(std::chrono::steady_clock::now().time_since_epoch().count());
    }
};
```

**Context Manager Component**
```cpp
#include <memory>
#include <vector>
#include <functional>

class ContextManager {
private:
    ExecutionContext base_context_;
    std::unique_ptr<ObjectPool<ExecutionContext>> context_pool_;
    std::vector<std::function<void(ExecutionContext&, const JsonEvent&)>> injectors_;
    
public:
    // Efficient context creation with object pooling
    std::unique_ptr<ExecutionContext> create_context(const JsonEvent& event) {
        auto context = context_pool_->acquire();
        
        // Reset to base state
        *context = base_context_;
        
        // Automatic injection of event and stream metadata
        for (const auto& injector : injectors_) {
            injector(*context, event);
        }
        
        return context;
    }
    
    // Pluggable metadata injection system
    void add_injector(std::function<void(ExecutionContext&, const JsonEvent&)> injector) {
        injectors_.emplace_back(std::move(injector));
    }
    
    // Type-safe metadata access setup
    void setup_default_injectors() {
        // Inject event metadata
        add_injector([](ExecutionContext& ctx, const JsonEvent& event) {
            nlohmann::json metadata;
            metadata["timestamp"] = event.timestamp;
            metadata["id"] = event.id;
            metadata["source"] = event.metadata.source;
            ctx.set_variable("event_metadata", metadata);
        });
        
        // Inject stream metadata
        add_injector([](ExecutionContext& ctx, const JsonEvent& event) {
            nlohmann::json stream_meta;
            stream_meta["pipeline"] = event.metadata.pipeline;
            stream_meta["routing_key"] = event.metadata.routing_key.value_or("");
            ctx.set_variable("stream_metadata", stream_meta);
        });
    }
};
```

#### AI Provider Layer (Reliability Focused)

**Provider Registry Component**
```cpp
#include <unordered_map>
#include <memory>
#include <future>
#include <chrono>

class ProviderRegistry {
private:
    std::unordered_map<ProviderId, std::unique_ptr<AIProvider>> providers_;
    std::unique_ptr<HealthMonitor> health_monitor_;
    std::unique_ptr<CostTracker> cost_tracker_;
    
public:
    // Intelligent request routing with health awareness
    std::future<AIResponse> send_request(
        const ProviderId& provider_id,
        const AIRequest& request) {
        
        return std::async(std::launch::async, [this, provider_id, request]() {
            auto provider_it = providers_.find(provider_id);
            if (provider_it == providers_.end()) {
                throw ProviderError("Provider not found: " + provider_id);
            }
            
            auto provider = provider_it->second.get();
            
            // Check provider health before request
            if (!health_monitor_->is_healthy(provider_id)) {
                throw ProviderError("Provider unhealthy: " + provider_id);
            }
            
            // Track cost before request
            auto estimated_cost = provider->estimate_cost(request);
            cost_tracker_->reserve_budget(provider_id, estimated_cost);
            
            try {
                // Automatic retry and failover logic
                auto response = provider->send_request(request);
                
                // Update actual cost
                auto actual_cost = provider->calculate_actual_cost(request, response);
                cost_tracker_->record_cost(provider_id, actual_cost);
                
                return response;
            } catch (const std::exception& e) {
                // Release reserved budget on failure
                cost_tracker_->release_budget(provider_id, estimated_cost);
                throw;
            }
        });
    }
    
    // Cost-aware provider selection
    std::optional<ProviderId> get_optimal_provider(const SelectionCriteria& criteria) {
        std::vector<std::pair<ProviderId, double>> scored_providers;
        
        for (const auto& [provider_id, provider] : providers_) {
            if (!health_monitor_->is_healthy(provider_id)) {
                continue;
            }
            
            double score = calculate_provider_score(provider_id, criteria);
            scored_providers.emplace_back(provider_id, score);
        }
        
        if (scored_providers.empty()) {
            return std::nullopt;
        }
        
        // Return provider with highest score
        auto best = std::max_element(scored_providers.begin(), scored_providers.end(),
            [](const auto& a, const auto& b) { return a.second < b.second; });
        
        return best->first;
    }
    
private:
    double calculate_provider_score(const ProviderId& provider_id, const SelectionCriteria& criteria) {
        double score = 0.0;
        
        // Factor in cost efficiency
        auto cost_efficiency = cost_tracker_->get_cost_efficiency(provider_id);
        score += criteria.cost_weight * cost_efficiency;
        
        // Factor in performance
        auto avg_latency = health_monitor_->get_average_latency(provider_id);
        score += criteria.performance_weight * (1.0 / avg_latency.count());
        
        // Factor in reliability
        auto success_rate = health_monitor_->get_success_rate(provider_id);
        score += criteria.reliability_weight * success_rate;
        
        return score;
    }
};
```

**Rate Limiter Component**
```cpp
#include <unordered_map>
#include <chrono>
#include <mutex>
#include <condition_variable>

class RateLimiter {
private:
    std::unordered_map<ProviderId, std::unique_ptr<TokenBucketLimiter>> limiters_;
    std::unique_ptr<GlobalTokenLimiter> global_limiter_;
    BackoffStrategy backoff_strategy_;
    std::mutex limiters_mutex_;
    
public:
    // Token-based rate limiting per provider
    std::future<void> acquire_tokens(const ProviderId& provider_id, uint32_t tokens) {
        return std::async(std::launch::async, [this, provider_id, tokens]() {
            std::unique_lock<std::mutex> lock(limiters_mutex_);
            
            auto limiter_it = limiters_.find(provider_id);
            if (limiter_it == limiters_.end()) {
                throw RateLimitError("No rate limiter configured for provider: " + provider_id);
            }
            
            auto limiter = limiter_it->second.get();
            lock.unlock();
            
            // Intelligent backoff and retry logic
            auto backoff_duration = std::chrono::milliseconds(100);
            while (!limiter->try_acquire(tokens)) {
                // Burst handling with configurable limits
                if (backoff_duration > std::chrono::seconds(30)) {
                    throw RateLimitError("Rate limit exceeded for provider: " + provider_id);
                }
                
                std::this_thread::sleep_for(backoff_duration);
                backoff_duration = backoff_strategy_.next_backoff(backoff_duration);
            }
        });
    }
    
    // Dynamic rate limit updates
    void update_limits(const ProviderId& provider_id, const RateLimit& new_limits) {
        std::unique_lock<std::mutex> lock(limiters_mutex_);
        
        auto limiter_it = limiters_.find(provider_id);
        if (limiter_it != limiters_.end()) {
            limiter_it->second->update_limits(new_limits);
        } else {
            limiters_[provider_id] = std::make_unique<TokenBucketLimiter>(new_limits);
        }
    }
    
    void add_provider(const ProviderId& provider_id, const RateLimit& limits) {
        std::unique_lock<std::mutex> lock(limiters_mutex_);
        limiters_[provider_id] = std::make_unique<TokenBucketLimiter>(limits);
    }
};
```

#### Management Layer (Operational Excellence)

**Configuration Manager Component**
```cpp
#include <memory>
#include <shared_mutex>
#include <functional>
#include <vector>

class ConfigurationManager {
private:
    mutable std::shared_mutex config_mutex_;
    std::shared_ptr<ColloquiumConfig> config_;
    std::unique_ptr<ConfigValidator> validator_;
    std::vector<std::function<void(const ColloquiumConfig&)>> change_listeners_;
    
public:
    // Atomic configuration updates with validation
    ConfigUpdateResult update_config(std::unique_ptr<ColloquiumConfig> new_config) {
        // Validate before applying
        auto validation_result = validator_->validate(*new_config);
        if (!validation_result.is_valid) {
            return {false, validation_result.error_message};
        }
        
        // Apply atomically
        {
            std::unique_lock<std::shared_mutex> lock(config_mutex_);
            config_ = std::shared_ptr<ColloquiumConfig>(new_config.release());
        }
        
        // Notify listeners
        notify_config_change(*config_);
        
        return {true, ""};
    }
    
    // Thread-safe configuration access
    std::shared_ptr<const ColloquiumConfig> get_config() const {
        std::shared_lock<std::shared_mutex> lock(config_mutex_);
        return config_;
    }
    
    // Real-time configuration change notifications
    void subscribe_to_changes(std::function<void(const ColloquiumConfig&)> listener) {
        change_listeners_.emplace_back(std::move(listener));
    }
    
private:
    void notify_config_change(const ColloquiumConfig& new_config) {
        for (const auto& listener : change_listeners_) {
            try {
                listener(new_config);
            } catch (const std::exception& e) {
                // Log error but continue notifying other listeners
                std::cerr << "Config change listener error: " << e.what() << std::endl;
            }
        }
    }
};
```

## Data Flow Architecture

### Event Processing Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Event     │    │   Event     │    │ Processing  │    │   Output    │
│  Ingestion  │───▶│   Router    │───▶│  Pipeline   │───▶│  Dispatcher │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                    │                  │                    │
      ▼                    ▼                  ▼                    ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Schema    │    │   Content   │    │  Computo    │    │   Sink      │
│ Validation  │    │   Analysis  │    │Transformation│    │ Connectors  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

#### Event Ingestion Stage
**Responsibilities:**
- Accept events from multiple source types (HTTP, Kafka, MQTT, etc.)
- Perform initial JSON schema validation
- Assign unique event IDs and timestamps
- Queue events for processing with backpressure handling

**C++ Implementation Pattern:**
```cpp
class EventIngestionStage {
private:
    std::unique_ptr<EventQueue> event_queue_;
    std::unique_ptr<SchemaValidator> schema_validator_;
    std::atomic<uint64_t> event_counter_{0};
    
public:
    void ingest_event(const std::string& raw_event) {
        // Parse JSON with error handling
        nlohmann::json event_data;
        try {
            event_data = nlohmann::json::parse(raw_event);
        } catch (const nlohmann::json::parse_error& e) {
            throw IngestionError("Invalid JSON: " + std::string(e.what()));
        }
        
        // Validate schema
        if (!schema_validator_->validate(event_data)) {
            throw IngestionError("Schema validation failed");
        }
        
        // Create event with metadata
        JsonEvent event;
        event.id = generate_event_id();
        event.timestamp = JsonEvent::current_timestamp();
        event.data = std::move(event_data);
        
        // Queue with backpressure handling
        if (!event_queue_->try_enqueue(std::move(event))) {
            throw BackpressureError("Event queue full");
        }
    }
    
private:
    std::string generate_event_id() {
        return "evt_" + std::to_string(event_counter_.fetch_add(1));
    }
};
```

#### Event Routing Stage
**Responsibilities:**
- Analyze JSON content for routing decisions
- Apply routing rules based on event structure and values
- Load balance events across processing pipelines
- Handle routing failures with dead letter queues

**C++ Implementation Pattern:**
```cpp
class ContentBasedRouter {
private:
    std::vector<std::unique_ptr<RoutingRule>> rules_;
    std::unique_ptr<LoadBalancer> load_balancer_;
    std::unique_ptr<DeadLetterQueue> dead_letter_queue_;
    
public:
    std::vector<RouteTarget> route_event(const JsonEvent& event) {
        std::vector<RouteTarget> targets;
        
        // Apply routing rules
        for (const auto& rule : rules_) {
            if (rule->matches(event)) {
                auto rule_targets = rule->get_targets();
                targets.insert(targets.end(), rule_targets.begin(), rule_targets.end());
            }
        }
        
        // Load balance if multiple targets
        if (targets.size() > 1) {
            targets = load_balancer_->balance(targets, event);
        }
        
        // Handle no targets found
        if (targets.empty()) {
            dead_letter_queue_->enqueue(event, "No routing targets found");
        }
        
        return targets;
    }
};
```

#### Processing Pipeline Stage
**Responsibilities:**
- Execute Computo transformation scripts
- Inject context metadata (timestamps, routing info, etc.)
- Handle transformation errors with configurable policies
- Collect performance metrics and execution traces

**C++ Implementation Pattern:**
```cpp
class ComputoProcessingStage : public ProcessingStage {
private:
    std::shared_ptr<ComputoScriptExecutor> script_executor_;
    std::string script_id_;
    ErrorPolicy error_policy_;
    
public:
    JsonEvent process(JsonEvent event) override {
        auto start_time = std::chrono::steady_clock::now();
        
        try {
            // Create execution context with metadata
            ExecutionContext context;
            inject_metadata(context, event);
            
            // Execute transformation
            auto future = script_executor_->execute_script(script_id_, std::move(event), context);
            auto result = future.get();
            
            // Record metrics
            auto duration = std::chrono::steady_clock::now() - start_time;
            record_execution_metrics(duration);
            
            return result;
        } catch (const computo::ComputoException& e) {
            return error_policy_.handle_error(std::move(event), e);
        }
    }
    
private:
    void inject_metadata(ExecutionContext& context, const JsonEvent& event) {
        nlohmann::json metadata;
        metadata["processing_time"] = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now().time_since_epoch()).count();
        metadata["stage"] = "computo_processing";
        context.set_variable("processing_metadata", metadata);
    }
};
```

### AI Provider Integration Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ AI Request  │    │  Provider   │    │ Rate Limit  │    │  Request    │
│ Generation  │───▶│  Selection  │───▶│   Check     │───▶│  Execution  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
      │                    │                  │                    │
      ▼                    ▼                  ▼                    ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Prompt    │    │    Cost     │    │   Token     │    │  Response   │
│ Templating  │    │ Estimation  │    │ Counting    │    │ Processing  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

#### AI Request Generation
**C++ Implementation Pattern:**
```cpp
class AIRequestGenerator {
private:
    std::unordered_map<std::string, PromptTemplate> templates_;
    std::unique_ptr<TemplateEngine> template_engine_;
    
public:
    AIRequest generate_request(const JsonEvent& event, const std::string& template_id) {
        auto template_it = templates_.find(template_id);
        if (template_it == templates_.end()) {
            throw AIRequestError("Template not found: " + template_id);
        }
        
        // Apply template with event data
        auto prompt = template_engine_->render(template_it->second, event.data);
        
        // Create request with metadata
        AIRequest request;
        request.prompt = prompt;
        request.model = template_it->second.preferred_model;
        request.max_tokens = template_it->second.max_tokens;
        request.temperature = template_it->second.temperature;
        
        // Estimate cost
        request.estimated_cost = estimate_cost(request);
        
        return request;
    }
    
private:
    double estimate_cost(const AIRequest& request) {
        // Token-based cost estimation
        auto input_tokens = count_tokens(request.prompt);
        auto output_tokens = request.max_tokens;
        
        // Provider-specific pricing
        auto provider_pricing = get_provider_pricing(request.model);
        return (input_tokens * provider_pricing.input_cost) + 
               (output_tokens * provider_pricing.output_cost);
    }
};
```

## Concurrency and Threading Model

### Thread Pool Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Main Event Loop                         │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │   Accept    │ │    Route    │ │       Monitor           │ │
│  │   Events    │ │   Events    │ │       Health            │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                Processing Thread Pool                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │   Worker    │ │   Worker    │ │       Worker            │ │
│  │   Thread 1  │ │   Thread 2  │ │       Thread N          │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                   I/O Thread Pool                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │  AI API     │ │   Storage   │ │      Network            │ │
│  │ Requests    │ │    I/O      │ │       I/O               │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Thread Safety Strategies

#### C++ Thread Safety Implementation
```cpp
class ThreadSafeEventProcessor {
private:
    // Immutable data structures
    std::shared_ptr<const ProcessingConfig> config_;
    
    // Thread-local storage for execution contexts
    thread_local std::unique_ptr<ExecutionContext> local_context_;
    
    // Lock-free queue for event passing
    moodycamel::ConcurrentQueue<JsonEvent> event_queue_;
    
    // Atomic counters for metrics
    std::atomic<uint64_t> events_processed_{0};
    std::atomic<uint64_t> events_failed_{0};
    
public:
    void process_events() {
        // Initialize thread-local context if needed
        if (!local_context_) {
            local_context_ = std::make_unique<ExecutionContext>();
        }
        
        JsonEvent event;
        while (event_queue_.try_dequeue(event)) {
            try {
                // Process with thread-local context
                auto result = process_single_event(std::move(event));
                events_processed_.fetch_add(1, std::memory_order_relaxed);
                
                // Forward to next stage
                forward_event(std::move(result));
            } catch (const std::exception& e) {
                events_failed_.fetch_add(1, std::memory_order_relaxed);
                handle_processing_error(std::move(event), e);
            }
        }
    }
    
private:
    JsonEvent process_single_event(JsonEvent event) {
        // Use thread-local context for isolation
        local_context_->reset();
        local_context_->set_input(event.data);
        
        // Execute transformation
        auto result = execute_transformation(*local_context_);
        
        // Create output event
        JsonEvent output = event;
        output.data = result;
        return output;
    }
};
```

## Storage Architecture

### Event Storage Design

```
┌─────────────────────────────────────────────────────────────┐
│                    Storage Layer                           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐ │
│  │  Memory     │ │    Disk     │ │       Remote            │ │
│  │  Buffer     │ │   Storage   │ │       Storage           │ │
│  └─────────────┘ └─────────────┘ └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

#### C++ Storage Implementation
```cpp
class HybridEventStorage {
private:
    // Memory buffer for hot data
    std::unique_ptr<RingBuffer<JsonEvent>> memory_buffer_;
    
    // Disk storage for persistence
    std::unique_ptr<AppendOnlyLog> disk_storage_;
    
    // Remote storage for archival
    std::unique_ptr<RemoteStorageClient> remote_storage_;
    
    // Background threads for data movement
    std::thread memory_to_disk_thread_;
    std::thread disk_to_remote_thread_;
    
public:
    EventId store_event(const JsonEvent& event) {
        // Store in memory buffer first
        auto event_id = memory_buffer_->append(event);
        
        // Schedule disk write
        disk_write_queue_.enqueue({event_id, event});
        
        return event_id;
    }
    
    std::optional<JsonEvent> retrieve_event(const EventId& event_id) {
        // Check memory buffer first
        if (auto event = memory_buffer_->get(event_id)) {
            return *event;
        }
        
        // Check disk storage
        if (auto event = disk_storage_->get(event_id)) {
            return *event;
        }
        
        // Check remote storage
        return remote_storage_->get(event_id);
    }
    
private:
    void process_disk_writes() {
        EventWithId item;
        while (disk_write_queue_.wait_dequeue(item)) {
            disk_storage_->append(item.id, item.event);
            
            // Schedule remote archival if needed
            if (should_archive(item.event)) {
                remote_archive_queue_.enqueue(item);
            }
        }
    }
};
```

## Error Handling and Recovery

### C++ Error Handling Implementation
```cpp
class ErrorHandler {
private:
    std::unordered_map<ErrorType, std::unique_ptr<ErrorPolicy>> policies_;
    std::unique_ptr<DeadLetterQueue> dead_letter_queue_;
    std::unique_ptr<CircuitBreaker> circuit_breaker_;
    
public:
    JsonEvent handle_error(JsonEvent event, const std::exception& error) {
        auto error_type = classify_error(error);
        
        auto policy_it = policies_.find(error_type);
        if (policy_it == policies_.end()) {
            // Default policy: send to dead letter queue
            dead_letter_queue_->enqueue(std::move(event), error.what());
            throw ProcessingError("No error policy configured for error type");
        }
        
        return policy_it->second->handle(std::move(event), error);
    }
    
private:
    ErrorType classify_error(const std::exception& error) {
        // Classify based on exception type and message
        if (dynamic_cast<const std::bad_alloc*>(&error)) {
            return ErrorType::RESOURCE_EXHAUSTION;
        }
        if (dynamic_cast<const std::system_error*>(&error)) {
            return ErrorType::SYSTEM_ERROR;
        }
        if (dynamic_cast<const computo::ComputoException*>(&error)) {
            return ErrorType::TRANSFORMATION_ERROR;
        }
        return ErrorType::UNKNOWN;
    }
};

class RetryErrorPolicy : public ErrorPolicy {
private:
    int max_retries_;
    std::chrono::milliseconds base_delay_;
    
public:
    JsonEvent handle(JsonEvent event, const std::exception& error) override {
        auto retry_count = event.metadata.retry_count;
        
        if (retry_count >= max_retries_) {
            throw ProcessingError("Max retries exceeded");
        }
        
        // Exponential backoff
        auto delay = base_delay_ * std::pow(2, retry_count);
        std::this_thread::sleep_for(delay);
        
        // Increment retry count
        event.metadata.retry_count++;
        
        return event;
    }
};
```

## Performance Optimization

### C++ Performance Optimizations

#### Zero-Copy JSON Processing
```cpp
class ZeroCopyJsonProcessor {
private:
    // Use string_view for zero-copy operations
    std::string_view find_json_field(std::string_view json_data, std::string_view field) {
        // Parse JSON without copying
        auto doc = simdjson::padded_string(json_data);
        auto parser = simdjson::dom::parser{};
        auto element = parser.parse(doc);
        
        // Return view into original data
        return element[field].get_string();
    }
    
    // Memory-mapped file for large JSON documents
    class MappedJsonFile {
    private:
        int fd_;
        void* mapped_data_;
        size_t file_size_;
        
    public:
        MappedJsonFile(const std::string& filename) {
            fd_ = open(filename.c_str(), O_RDONLY);
            if (fd_ == -1) {
                throw std::system_error(errno, std::system_category());
            }
            
            struct stat st;
            if (fstat(fd_, &st) == -1) {
                close(fd_);
                throw std::system_error(errno, std::system_category());
            }
            
            file_size_ = st.st_size;
            mapped_data_ = mmap(nullptr, file_size_, PROT_READ, MAP_PRIVATE, fd_, 0);
            if (mapped_data_ == MAP_FAILED) {
                close(fd_);
                throw std::system_error(errno, std::system_category());
            }
        }
        
        std::string_view data() const {
            return std::string_view(static_cast<const char*>(mapped_data_), file_size_);
        }
    };
};
```

#### SIMD Optimizations
```cpp
class SIMDJsonProcessor {
public:
    // Vectorized JSON field extraction
    std::vector<std::string> extract_fields_vectorized(
        const std::vector<std::string>& json_documents,
        const std::string& field_name) {
        
        std::vector<std::string> results;
        results.reserve(json_documents.size());
        
        // Process in batches for SIMD efficiency
        constexpr size_t batch_size = 8;
        for (size_t i = 0; i < json_documents.size(); i += batch_size) {
            auto batch_end = std::min(i + batch_size, json_documents.size());
            
            // Use simdjson for vectorized parsing
            simdjson::dom::parser parser;
            for (size_t j = i; j < batch_end; ++j) {
                auto element = parser.parse(json_documents[j]);
                results.emplace_back(element[field_name].get_string());
            }
        }
        
        return results;
    }
};
```

## Security Architecture

### C++ Security Implementation
```cpp
class SecurityManager {
private:
    std::unique_ptr<Encryptor> encryptor_;
    std::unique_ptr<AuthenticationManager> auth_manager_;
    std::unique_ptr<AuditLogger> audit_logger_;
    
public:
    // Encrypt sensitive data at rest
    std::string encrypt_data(const std::string& plaintext, const std::string& key_id) {
        auto key = get_encryption_key(key_id);
        return encryptor_->encrypt(plaintext, key);
    }
    
    // Authenticate API requests
    bool authenticate_request(const HttpRequest& request) {
        auto auth_header = request.get_header("Authorization");
        if (!auth_header) {
            return false;
        }
        
        auto result = auth_manager_->verify_token(*auth_header);
        
        // Log authentication attempt
        audit_logger_->log_authentication_attempt(
            request.get_remote_ip(),
            result.user_id,
            result.success
        );
        
        return result.success;
    }
    
private:
    EncryptionKey get_encryption_key(const std::string& key_id) {
        // Integrate with key management service
        return key_manager_->get_key(key_id);
    }
};
```

## Monitoring and Observability

### C++ Monitoring Implementation
```cpp
class MetricsCollector {
private:
    // Lock-free metrics collection
    std::atomic<uint64_t> events_processed_{0};
    std::atomic<uint64_t> events_failed_{0};
    
    // Histogram for latency tracking
    std::unique_ptr<prometheus::Histogram> latency_histogram_;
    
    // Gauge for active connections
    std::unique_ptr<prometheus::Gauge> active_connections_;
    
public:
    void record_event_processed() {
        events_processed_.fetch_add(1, std::memory_order_relaxed);
    }
    
    void record_processing_latency(std::chrono::milliseconds latency) {
        latency_histogram_->Observe(latency.count());
    }
    
    void set_active_connections(int count) {
        active_connections_->Set(count);
    }
    
    // Export metrics for Prometheus
    std::string export_metrics() {
        auto registry = prometheus::Registry::create();
        
        // Add all metrics to registry
        registry->Add(latency_histogram_);
        registry->Add(active_connections_);
        
        // Add atomic counters
        auto counter_family = prometheus::BuildCounter()
            .Name("events_total")
            .Help("Total number of events processed")
            .Register(*registry);
        
        counter_family.Add({{"status", "processed"}}).Set(events_processed_.load());
        counter_family.Add({{"status", "failed"}}).Set(events_failed_.load());
        
        // Serialize to Prometheus format
        auto serializer = prometheus::TextSerializer();
        return serializer.Serialize(registry->Collect());
    }
};
```

## Configuration Management

### C++ Configuration System
```cpp
class ConfigurationSystem {
private:
    std::shared_ptr<yaml::Node> config_;
    std::shared_mutex config_mutex_;
    std::vector<std::function<void(const yaml::Node&)>> change_callbacks_;
    
public:
    // Load configuration from YAML file
    void load_configuration(const std::string& config_path) {
        try {
            auto new_config = yaml::LoadFile(config_path);
            validate_configuration(new_config);
            
            {
                std::unique_lock<std::shared_mutex> lock(config_mutex_);
                config_ = std::make_shared<yaml::Node>(new_config);
            }
            
            notify_configuration_change(*config_);
        } catch (const yaml::Exception& e) {
            throw ConfigurationError("Failed to load configuration: " + std::string(e.what()));
        }
    }
    
    // Hot reload configuration
    void reload_configuration() {
        // Use file watcher for automatic reload
        auto watcher = std::make_unique<FileWatcher>(config_path_);
        watcher->on_change([this](const std::string& path) {
            try {
                load_configuration(path);
            } catch (const ConfigurationError& e) {
                // Log error but don't crash
                std::cerr << "Configuration reload failed: " << e.what() << std::endl;
            }
        });
    }
    
    // Thread-safe configuration access
    template<typename T>
    T get_config_value(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(config_mutex_);
        return config_->at(key).as<T>();
    }
    
private:
    void validate_configuration(const yaml::Node& config) {
        // Validate required fields
        if (!config["pipelines"]) {
            throw ConfigurationError("Missing required field: pipelines");
        }
        
        // Validate pipeline configurations
        for (const auto& pipeline : config["pipelines"]) {
            validate_pipeline_config(pipeline);
        }
    }
    
    void validate_pipeline_config(const yaml::Node& pipeline) {
        if (!pipeline["name"]) {
            throw ConfigurationError("Pipeline missing required field: name");
        }
        
        if (!pipeline["sources"]) {
            throw ConfigurationError("Pipeline missing required field: sources");
        }
        
        if (!pipeline["transforms"]) {
            throw ConfigurationError("Pipeline missing required field: transforms");
        }
        
        if (!pipeline["sinks"]) {
            throw ConfigurationError("Pipeline missing required field: sinks");
        }
    }
};
```

## Success Criteria

### Technical Metrics
- **Throughput**: 10,000+ events/second single node, 100,000+ events/second cluster
- **Latency**: Sub-100ms end-to-end processing for 95th percentile
- **Memory Efficiency**: <2GB memory for basic workloads, <10% CPU overhead
- **Reliability**: 99.9% single node, 99.99% multi-node cluster availability

### C++ Implementation Quality
- **Type Safety**: Extensive use of strong types and RAII
- **Performance**: Zero-copy operations and lock-free algorithms where possible
- **Memory Safety**: Smart pointers and automatic resource management
- **Thread Safety**: Immutable data structures and thread-local storage

### Operational Excellence
- **Single Binary**: Self-contained deployment with minimal dependencies
- **Hot Reload**: Configuration changes without service interruption
- **Observability**: Comprehensive metrics and distributed tracing
- **Error Recovery**: Automatic recovery from 90%+ of transient failures

This architecture leverages modern C++ features and best practices to create a high-performance, reliable, and maintainable stream processing platform. The design emphasizes simplicity, performance, and operational excellence while avoiding the over-engineering pitfalls common in distributed systems.