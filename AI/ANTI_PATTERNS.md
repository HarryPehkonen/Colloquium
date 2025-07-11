# ANTI_PATTERNS.md

## Colloquium Anti-Patterns: What NOT to Do

This document catalogs specific anti-patterns to avoid when implementing Colloquium, drawing lessons from the over-engineering discovered in Computo's development and the common pitfalls of stream processing platforms. These patterns represent real traps that lead to complexity, poor performance, and operational nightmares.

## Core Anti-Pattern Categories

### 1. Stream Processing Over-Engineering
### 2. AI Integration Complexity Traps  
### 3. Configuration and Management Bloat
### 4. Performance Optimization Premature Complexity
### 5. Monitoring and Observability Over-Instrumentation
### 6. Security and Authentication Over-Engineering
### 7. Deployment and Operations Complexity

## 1. Stream Processing Over-Engineering Anti-Patterns

### ❌ BAD: Complex Event Sourcing Implementation
**Problem**: Implementing full event sourcing when simple event storage is sufficient

```cpp
// DON'T DO THIS - Over-engineered event sourcing
#include <unordered_map>
#include <vector>
#include <memory>
#include <functional>

class EventSourcingEngine {
private:
    std::unique_ptr<EventStore> event_store_;
    std::unique_ptr<AggregateRepository> aggregate_repository_;
    std::unordered_map<std::string, std::function<std::vector<Event>(const Command&, const Aggregate&)>> command_handlers_;
    std::unordered_map<std::string, std::function<void(const Event&)>> event_handlers_;
    std::unique_ptr<SagaOrchestrator> saga_orchestrator_;
    std::unique_ptr<SnapshotStore> snapshot_store_;
    std::unique_ptr<ProjectionManager> projection_manager_;
    std::unordered_map<std::string, std::unique_ptr<ViewMaterializer>> view_materializers_;
    
public:
    std::vector<Event> handle_command(const Command& command) {
        // Load aggregate from event stream
        auto aggregate = aggregate_repository_->load(command.aggregate_id);
        
        // Apply command to aggregate
        auto handler_it = command_handlers_.find(command.command_type);
        if (handler_it == command_handlers_.end()) {
            throw std::runtime_error("Handler not found for command type");
        }
        
        auto events = handler_it->second(command, *aggregate);
        
        // Store events
        event_store_->append_events(command.aggregate_id, events);
        
        // Update projections
        for (const auto& event : events) {
            projection_manager_->apply_event(event);
        }
        
        // Check for saga triggers
        saga_orchestrator_->process_events(events);
        
        return events;
    }
    
    // Complex aggregate reconstruction
    std::unique_ptr<Aggregate> reconstruct_aggregate(const std::string& aggregate_id) {
        auto events = event_store_->get_events(aggregate_id);
        auto aggregate = std::make_unique<Aggregate>(aggregate_id);
        
        // Apply all events to rebuild state
        for (const auto& event : events) {
            aggregate->apply_event(event);
        }
        
        return aggregate;
    }
    
    // Snapshot management
    void create_snapshot(const std::string& aggregate_id) {
        auto aggregate = reconstruct_aggregate(aggregate_id);
        auto snapshot = create_snapshot_from_aggregate(*aggregate);
        snapshot_store_->save_snapshot(aggregate_id, snapshot);
    }
};
```

**Problems**:
- Massive complexity for simple event processing needs
- Multiple abstractions (aggregates, sagas, projections) that aren't needed
- Performance overhead from complex state reconstruction
- Difficult to debug and maintain

**✅ GOOD: Simple Event Processing**
```cpp
// Simple, effective event processing
class StreamProcessor {
private:
    std::unique_ptr<EventStorage> storage_;
    std::vector<std::unique_ptr<EventTransformer>> transformers_;
    std::vector<std::unique_ptr<EventSink>> sinks_;
    
public:
    void process_event(const JsonEvent& event) {
        // Store event for replay capability
        storage_->store(event);
        
        // Apply transformations
        JsonEvent transformed_event = event;
        for (auto& transformer : transformers_) {
            transformed_event = transformer->transform(transformed_event);
        }
        
        // Send to sinks
        for (auto& sink : sinks_) {
            sink->send(transformed_event);
        }
    }
};
```

### ❌ BAD: Complex Stream Partitioning Schemes
**Problem**: Over-engineering partitioning when simple schemes work

```cpp
// DON'T DO THIS - Complex multi-level partitioning
class AdvancedPartitioner {
private:
    std::unique_ptr<PartitionStrategy> primary_strategy_;
    std::vector<std::unique_ptr<PartitionStrategy>> secondary_strategies_;
    std::unique_ptr<ConsistentHashRing> load_balancer_;
    std::unique_ptr<PartitionRebalancer> partition_rebalancer_;
    std::unique_ptr<HotSpotDetector> hot_spot_detector_;
    std::unique_ptr<PartitionMerger> partition_merger_;
    
    struct PartitionAssignment {
        enum Type { Single, Multi } type;
        std::vector<uint32_t> partitions;
    };
    
public:
    PartitionAssignment partition_event(const JsonEvent& event) {
        // Primary partitioning
        uint32_t primary_partition = primary_strategy_->partition(event);
        
        // Check for hot spots
        if (hot_spot_detector_->is_hot_spot(primary_partition)) {
            // Apply secondary partitioning strategies
            for (auto& strategy : secondary_strategies_) {
                auto secondary_partition = strategy->partition(event);
                if (secondary_partition != primary_partition) {
                    return PartitionAssignment{PartitionAssignment::Multi, 
                                             {primary_partition, secondary_partition}};
                }
            }
        }
        
        // Apply load balancing
        uint32_t balanced_partition = load_balancer_->balance(primary_partition);
        
        // Check if rebalancing is needed
        if (partition_rebalancer_->should_rebalance()) {
            balanced_partition = partition_rebalancer_->rebalance(balanced_partition);
        }
        
        return PartitionAssignment{PartitionAssignment::Single, {balanced_partition}};
    }
    
    void handle_partition_merge(const std::vector<uint32_t>& partitions) {
        partition_merger_->merge_partitions(partitions);
    }
};
```

**Problems**:
- Complex logic that's hard to reason about
- Multiple partitioning strategies create confusion
- Hot spot detection adds overhead to every event
- Load balancing complexity often unnecessary

**✅ GOOD: Simple Consistent Partitioning**
```cpp
// Simple, predictable partitioning
class SimplePartitioner {
private:
    size_t num_partitions_;
    
public:
    SimplePartitioner(size_t num_partitions) : num_partitions_(num_partitions) {}
    
    uint32_t partition_event(const JsonEvent& event) {
        // Use event ID for consistent partitioning
        std::hash<std::string> hasher;
        return static_cast<uint32_t>(hasher(event.id) % num_partitions_);
    }
};
```

### ❌ BAD: Over-Abstracted Pipeline Definitions
**Problem**: Creating complex DSLs when YAML configuration is sufficient

```cpp
// DON'T DO THIS - Complex pipeline DSL builder
template<typename T>
class PipelineBuilder {
private:
    std::vector<std::unique_ptr<PipelineStage>> stages_;
    std::unordered_map<std::string, std::unique_ptr<ErrorHandler>> error_handlers_;
    std::unordered_map<std::string, RetryPolicy> retry_policies_;
    std::unordered_map<std::string, std::unique_ptr<CircuitBreaker>> circuit_breakers_;
    std::unordered_map<std::string, std::unique_ptr<MetricsCollector>> metrics_collectors_;
    
public:
    template<typename Source>
    PipelineBuilder& source(Source&& source) {
        stages_.emplace_back(std::make_unique<SourceStage<Source>>(std::forward<Source>(source)));
        return *this;
    }
    
    template<typename Transform>
    PipelineBuilder& transform(Transform&& transform) {
        stages_.emplace_back(std::make_unique<TransformStage<Transform>>(std::forward<Transform>(transform)));
        return *this;
    }
    
    template<typename Filter>
    PipelineBuilder& filter(Filter&& filter) {
        stages_.emplace_back(std::make_unique<FilterStage<Filter>>(std::forward<Filter>(filter)));
        return *this;
    }
    
    template<typename ErrorHandler>
    PipelineBuilder& with_error_handler(const std::string& stage_id, ErrorHandler&& handler) {
        error_handlers_[stage_id] = std::make_unique<ErrorHandlerWrapper<ErrorHandler>>(
            std::forward<ErrorHandler>(handler));
        return *this;
    }
    
    PipelineBuilder& with_retry_policy(const std::string& stage_id, const RetryPolicy& policy) {
        retry_policies_[stage_id] = policy;
        return *this;
    }
    
    template<typename CircuitBreakerConfig>
    PipelineBuilder& with_circuit_breaker(const std::string& stage_id, 
                                         const CircuitBreakerConfig& config) {
        circuit_breakers_[stage_id] = std::make_unique<CircuitBreaker>(config);
        return *this;
    }
    
    template<typename MetricsConfig>
    PipelineBuilder& with_metrics(const std::string& stage_id, const MetricsConfig& config) {
        metrics_collectors_[stage_id] = std::make_unique<MetricsCollector>(config);
        return *this;
    }
    
    // ... 20+ more configuration methods
    
    std::unique_ptr<Pipeline> build() {
        validate_configuration();
        return std::make_unique<Pipeline>(std::move(stages_), std::move(error_handlers_),
                                        std::move(retry_policies_), std::move(circuit_breakers_),
                                        std::move(metrics_collectors_));
    }
};
```

**Problems**:
- Complex builder pattern harder to understand than YAML
- Type safety at expense of simplicity
- Difficult to version control and review changes
- Overloads developers with too many options

**✅ GOOD: Simple YAML Configuration**
```cpp
// Simple configuration loading
struct PipelineConfig {
    std::string name;
    
    struct SourceConfig {
        std::string type;
        std::unordered_map<std::string, std::string> config;
    } source;
    
    std::vector<std::string> transforms;
    
    struct SinkConfig {
        std::string type;
        std::unordered_map<std::string, std::string> config;
    } sink;
    
    static PipelineConfig load_from_yaml(const std::string& yaml_content) {
        // Use a simple YAML parser like yaml-cpp
        YAML::Node config = YAML::Load(yaml_content);
        
        PipelineConfig pipeline_config;
        pipeline_config.name = config["name"].as<std::string>();
        
        // Load source
        auto source_node = config["source"];
        pipeline_config.source.type = source_node["type"].as<std::string>();
        for (auto it : source_node["config"]) {
            pipeline_config.source.config[it.first.as<std::string>()] = it.second.as<std::string>();
        }
        
        // Load transforms
        for (auto transform : config["transforms"]) {
            pipeline_config.transforms.push_back(transform.as<std::string>());
        }
        
        // Load sink
        auto sink_node = config["sink"];
        pipeline_config.sink.type = sink_node["type"].as<std::string>();
        for (auto it : sink_node["config"]) {
            pipeline_config.sink.config[it.first.as<std::string>()] = it.second.as<std::string>();
        }
        
        return pipeline_config;
    }
};

// Example YAML:
/*
name: "ai-processing"
source:
  type: "http"
  config:
    port: "8080"
    path: "/events"
transforms:
  - '["obj", ["ai_request", ["get", ["$input"], "/request"]], ["metadata", ["$metadata"]]]'
sink:
  type: "ai_provider" 
  config:
    provider: "openai"
    model: "gpt-4"
*/
```

## 2. AI Integration Complexity Traps

### ❌ BAD: Over-Engineered Provider Abstraction
**Problem**: Creating complex abstractions that hide important provider differences

```cpp
// DON'T DO THIS - Over-abstracted AI provider interface
class UniversalAIProvider {
public:
    virtual ~UniversalAIProvider() = default;
    
    virtual std::future<TextResponse> generate_text(const TextGenerationRequest& request) = 0;
    virtual std::future<ImageResponse> generate_image(const ImageGenerationRequest& request) = 0;
    virtual std::future<AudioResponse> generate_audio(const AudioGenerationRequest& request) = 0;
    virtual std::future<SentimentScore> analyze_sentiment(const std::string& text) = 0;
    virtual std::future<std::vector<Entity>> extract_entities(const std::string& text) = 0;
    virtual std::future<std::string> translate_text(const std::string& text, const std::string& target_lang) = 0;
    virtual std::future<std::string> summarize_text(const std::string& text, size_t max_length) = 0;
    virtual std::future<ContentClassification> classify_content(const std::string& content) = 0;
    
    // Provider-specific capabilities
    virtual bool supports_streaming() const = 0;
    virtual bool supports_function_calling() const = 0;
    virtual bool supports_vision() const = 0;
    virtual size_t get_context_length() const = 0;
    virtual std::vector<std::string> get_supported_languages() const = 0;
    virtual RateLimit get_rate_limits() const = 0;
};

// Complex request routing based on capabilities
class ProviderOrchestrator {
private:
    std::unordered_map<std::string, std::unique_ptr<UniversalAIProvider>> providers_;
    std::unique_ptr<CapabilityMatrix> capability_matrix_;
    std::unique_ptr<CapabilityAwareLoadBalancer> load_balancer_;
    std::unordered_map<RequestType, std::vector<std::string>> fallback_chains_;
    
public:
    std::future<AIResponse> route_request(const AIRequest& request) {
        // Determine required capabilities
        auto required_caps = analyze_request_capabilities(request);
        
        // Find providers that support required capabilities
        auto capable_providers = capability_matrix_->find_providers(required_caps);
        
        if (capable_providers.empty()) {
            throw std::runtime_error("No providers support required capabilities");
        }
        
        // Apply capability-aware load balancing
        auto selected_provider = load_balancer_->select_provider(capable_providers, request);
        
        // Route request to selected provider
        return route_to_provider(selected_provider, request);
    }
    
private:
    std::vector<Capability> analyze_request_capabilities(const AIRequest& request) {
        // Complex capability analysis
        std::vector<Capability> capabilities;
        
        if (request.has_images()) capabilities.push_back(Capability::Vision);
        if (request.requires_streaming()) capabilities.push_back(Capability::Streaming);
        if (request.has_function_calls()) capabilities.push_back(Capability::FunctionCalling);
        
        return capabilities;
    }
};
```

**Problems**:
- Forces all providers to implement features they don't support
- Hides important provider-specific optimizations and features
- Complex capability detection and routing logic
- Interface becomes lowest common denominator

**✅ GOOD: Provider-Specific Implementations**
```cpp
// Simple, provider-specific interfaces that expose real capabilities
class OpenAIProvider {
public:
    std::future<nlohmann::json> chat_completion(const nlohmann::json& request) {
        return std::async(std::launch::async, [this, request]() {
            return send_openai_request("/chat/completions", request);
        });
    }
    
    std::future<nlohmann::json> embeddings(const std::vector<std::string>& texts) {
        return std::async(std::launch::async, [this, texts]() {
            nlohmann::json request = {
                {"model", "text-embedding-ada-002"},
                {"input", texts}
            };
            return send_openai_request("/embeddings", request);
        });
    }
    
private:
    nlohmann::json send_openai_request(const std::string& endpoint, const nlohmann::json& data);
};

class AnthropicProvider {
public:
    std::future<nlohmann::json> messages(const nlohmann::json& request) {
        return std::async(std::launch::async, [this, request]() {
            return send_anthropic_request("/v1/messages", request);
        });
    }
    
private:
    nlohmann::json send_anthropic_request(const std::string& endpoint, const nlohmann::json& data);
};

// Simple registry without complex abstraction
class ProviderRegistry {
private:
    std::unique_ptr<OpenAIProvider> openai_;
    std::unique_ptr<AnthropicProvider> anthropic_;
    std::unique_ptr<GoogleAIProvider> google_;
    
public:
    std::future<nlohmann::json> route_request(const nlohmann::json& request) {
        std::string provider = request.value("provider", "openai");
        
        if (provider == "openai" && openai_) {
            return openai_->chat_completion(request);
        } else if (provider == "anthropic" && anthropic_) {
            return anthropic_->messages(request);
        } else {
            throw std::runtime_error("Unsupported provider: " + provider);
        }
    }
};
```

### ❌ BAD: Complex Cost Optimization Algorithms
**Problem**: Over-engineering cost optimization when simple rules work better

```cpp
// DON'T DO THIS - Complex cost optimization
class AdvancedCostOptimizer {
private:
    std::unique_ptr<CostHistoryAnalyzer> historical_data_;
    std::unique_ptr<MLCostPredictor> ml_predictor_;
    std::unique_ptr<GeneticOptimizer> genetic_algorithm_;
    std::unique_ptr<QLearningAgent> reinforcement_learner_;
    std::unique_ptr<ParetoOptimizer> multi_objective_optimizer_;
    
    struct OptimizationResult {
        std::string provider_id;
        nlohmann::json optimized_request;
        double estimated_cost;
        double confidence_score;
    };
    
public:
    OptimizationResult optimize_request(const nlohmann::json& request) {
        // Analyze historical patterns
        auto patterns = historical_data_->analyze_similar_requests(request);
        
        // Predict costs for different providers using ML
        auto cost_predictions = ml_predictor_->predict_costs(request, patterns);
        
        // Apply genetic algorithm for multi-constraint optimization
        auto genetic_result = genetic_algorithm_->optimize(
            cost_predictions,
            extract_constraints(request),
            100 // generations
        );
        
        // Use reinforcement learning to adjust based on historical performance
        auto state_vector = create_state_vector(request, genetic_result);
        auto rl_adjustment = reinforcement_learner_->get_action(state_vector);
        
        // Apply Pareto optimization for multi-objective trade-offs
        std::vector<OptimizationCandidate> candidates = {genetic_result, rl_adjustment};
        std::vector<Objective> objectives = {
            Objective::Cost, Objective::Latency, Objective::Quality
        };
        
        auto pareto_optimal = multi_objective_optimizer_->find_pareto_front(candidates, objectives);
        
        return pareto_optimal.best_solution;
    }
    
private:
    std::vector<Constraint> extract_constraints(const nlohmann::json& request) {
        // Complex constraint analysis
        std::vector<Constraint> constraints;
        
        if (request.contains("max_cost")) {
            constraints.emplace_back(ConstraintType::MaxCost, request["max_cost"]);
        }
        if (request.contains("max_latency")) {
            constraints.emplace_back(ConstraintType::MaxLatency, request["max_latency"]);
        }
        if (request.contains("min_quality")) {
            constraints.emplace_back(ConstraintType::MinQuality, request["min_quality"]);
        }
        
        return constraints;
    }
};
```

**Problems**:
- Massive complexity for marginal cost savings
- Machine learning overkill for predictable pricing models
- Genetic algorithms unnecessary for simple optimization problems
- Hard to debug when optimization goes wrong

**✅ GOOD: Simple Rule-Based Cost Optimization**
```cpp
// Simple, effective cost optimization
class CostOptimizer {
private:
    struct ProviderCosts {
        double cost_per_input_token;
        double cost_per_output_token;
        uint32_t max_tokens;
    };
    
    std::unordered_map<std::string, ProviderCosts> provider_costs_;
    double daily_budget_;
    double current_cost_ = 0.0;
    
public:
    CostOptimizer(double daily_budget) : daily_budget_(daily_budget) {
        // Initialize provider costs
        provider_costs_["openai-gpt-4"] = {0.00003, 0.00006, 8192};
        provider_costs_["openai-gpt-3.5-turbo"] = {0.0000015, 0.000002, 4096};
        provider_costs_["anthropic-claude-3-haiku"] = {0.000015, 0.000075, 200000};
    }
    
    std::string select_provider(const nlohmann::json& request) {
        uint32_t estimated_tokens = estimate_token_count(request);
        
        // Find cheapest provider that meets requirements
        std::vector<std::pair<std::string, double>> candidates;
        
        for (const auto& [provider_id, costs] : provider_costs_) {
            // Check if provider supports required token count
            if (estimated_tokens > costs.max_tokens) {
                continue;
            }
            
            double estimated_cost = (estimated_tokens * costs.cost_per_input_token) +
                                   (estimated_tokens * 0.5 * costs.cost_per_output_token); // Assume 50% output ratio
            
            // Check budget constraints
            if (current_cost_ + estimated_cost > daily_budget_) {
                continue;
            }
            
            candidates.emplace_back(provider_id, estimated_cost);
        }
        
        if (candidates.empty()) {
            throw std::runtime_error("No viable provider within budget");
        }
        
        // Return cheapest option
        std::sort(candidates.begin(), candidates.end(), 
                  [](const auto& a, const auto& b) { return a.second < b.second; });
        
        return candidates[0].first;
    }
    
    void record_actual_cost(double cost) {
        current_cost_ += cost;
    }
    
private:
    uint32_t estimate_token_count(const nlohmann::json& request) {
        // Simple estimation: 4 characters = 1 token
        std::string content = request.dump();
        return static_cast<uint32_t>(content.length() / 4);
    }
};
```

### ❌ BAD: Complex Request/Response Transformation Pipelines
**Problem**: Building complex transformation chains when Computo can handle it

```cpp
// DON'T DO THIS - Complex transformation pipeline
class RequestTransformationPipeline {
private:
    std::unordered_map<std::string, std::unique_ptr<SchemaValidator>> schema_validators_;
    std::unordered_map<std::string, std::unique_ptr<FieldMapper>> field_mappers_;
    std::unordered_map<std::string, std::unique_ptr<TypeConverter>> type_converters_;
    std::unordered_map<std::string, std::unique_ptr<FormatTransformer>> format_transformers_;
    std::unique_ptr<ValidationPipeline> validation_pipeline_;
    std::unique_ptr<SanitizationPipeline> sanitization_pipeline_;
    
    struct TransformationResult {
        nlohmann::json transformed_request;
        std::vector<std::string> warnings;
        std::vector<std::string> validation_errors;
    };
    
public:
    TransformationResult transform_request(const nlohmann::json& universal_request,
                                         const std::string& target_provider) {
        TransformationResult result;
        
        // Validate input schema
        auto validator = schema_validators_.find(target_provider);
        if (validator == schema_validators_.end()) {
            throw std::runtime_error("Validator not found for provider: " + target_provider);
        }
        
        auto validation_result = validator->second->validate(universal_request);
        if (!validation_result.is_valid) {
            result.validation_errors = validation_result.errors;
            return result;
        }
        
        // Map fields
        auto field_mapper = field_mappers_.find(target_provider);
        if (field_mapper == field_mappers_.end()) {
            throw std::runtime_error("Field mapper not found for provider: " + target_provider);
        }
        
        auto mapped_request = field_mapper->second->map_fields(universal_request);
        
        // Convert types
        auto type_converter = type_converters_.find(target_provider);
        if (type_converter == type_converters_.end()) {
            throw std::runtime_error("Type converter not found for provider: " + target_provider);
        }
        
        auto converted_request = type_converter->second->convert_types(mapped_request);
        
        // Transform format
        auto format_transformer = format_transformers_.find(target_provider);
        if (format_transformer == format_transformers_.end()) {
            throw std::runtime_error("Format transformer not found for provider: " + target_provider);
        }
        
        auto formatted_request = format_transformer->second->transform_format(converted_request);
        
        // Run validation pipeline
        auto validated_request = validation_pipeline_->validate(formatted_request);
        result.warnings.insert(result.warnings.end(), 
                              validated_request.warnings.begin(), validated_request.warnings.end());
        
        // Run sanitization pipeline
        auto sanitized_request = sanitization_pipeline_->sanitize(validated_request.data);
        
        result.transformed_request = sanitized_request;
        return result;
    }
};
```

**Problems**:
- Multiple transformation layers when one would suffice
- Complex error handling across multiple pipeline stages
- Performance overhead from multiple data transformations
- Difficult to debug transformation failures

**✅ GOOD: Simple Computo Transformations**
```cpp
// Use Computo for all transformations - it's designed for this!
class SimpleTransformer {
private:
    std::unordered_map<std::string, std::string> computo_scripts_;
    
public:
    SimpleTransformer() {
        // Initialize transformation scripts for each provider
        computo_scripts_["openai"] = R"([
            "obj",
            ["model", ["get", ["$input"], "/model"]],
            ["messages", ["map", ["get", ["$input"], "/messages"], 
                ["lambda", ["msg"], 
                    ["obj", 
                        ["role", ["get", ["$", "/msg"], "/role"]],
                        ["content", ["get", ["$", "/msg"], "/content"]]
                    ]
                ]
            ]],
            ["max_tokens", ["get", ["$input"], "/max_tokens"]]
        ])";
        
        computo_scripts_["anthropic"] = R"([
            "obj",
            ["model", ["get", ["$input"], "/model"]],
            ["messages", ["get", ["$input"], "/messages"]],
            ["max_tokens", ["get", ["$input"], "/max_tokens"]],
            ["system", ["get", ["$input"], "/system"]]
        ])";
    }
    
    nlohmann::json transform_request(const nlohmann::json& request, 
                                   const std::string& target_provider) {
        auto script_it = computo_scripts_.find(target_provider);
        if (script_it == computo_scripts_.end()) {
            throw std::runtime_error("No transformation script for provider: " + target_provider);
        }
        
        // Parse and execute Computo script
        nlohmann::json script = nlohmann::json::parse(script_it->second);
        return computo::execute(script, request);
    }
    
    void add_transformation(const std::string& provider_id, const std::string& script) {
        // Validate script by parsing it
        nlohmann::json::parse(script); // Will throw if invalid
        computo_scripts_[provider_id] = script;
    }
};
```

## 3. Configuration and Management Bloat

### ❌ BAD: Over-Engineered Configuration Management
**Problem**: Building complex configuration systems when simple YAML works

```cpp
// DON'T DO THIS - Complex configuration framework
class ConfigurationFramework {
private:
    std::unordered_map<std::string, std::unique_ptr<ConfigLoader>> loaders_;
    std::vector<std::unique_ptr<ConfigValidator>> validators_;
    std::vector<std::unique_ptr<ConfigTransformer>> transformers_;
    std::unordered_map<std::string, std::unique_ptr<ConfigWatcher>> watchers_;
    std::unique_ptr<ConfigVersionManager> version_manager_;
    std::unique_ptr<ConfigMigrationEngine> migration_engine_;
    std::unique_ptr<ConfigTemplateEngine> template_engine_;
    std::unique_ptr<ConfigEncryptionManager> encryption_manager_;
    
    struct ConfigurationResult {
        nlohmann::json config;
        std::vector<std::string> warnings;
        std::vector<std::string> migrations_applied;
        bool requires_restart;
    };
    
public:
    ConfigurationResult load_configuration(const std::string& config_path) {
        ConfigurationResult result;
        
        // Detect configuration format
        std::string format = detect_format(config_path);
        
        // Load using appropriate loader
        auto loader_it = loaders_.find(format);
        if (loader_it == loaders_.end()) {
            throw std::runtime_error("Unsupported configuration format: " + format);
        }
        
        nlohmann::json raw_config = loader_it->second->load(config_path);
        
        // Apply transformations
        nlohmann::json config = raw_config;
        for (auto& transformer : transformers_) {
            auto transform_result = transformer->transform(config);
            config = transform_result.transformed_config;
            result.warnings.insert(result.warnings.end(),
                                 transform_result.warnings.begin(),
                                 transform_result.warnings.end());
        }
        
        // Process templates
        auto template_result = template_engine_->process_templates(config);
        config = template_result.processed_config;
        result.warnings.insert(result.warnings.end(),
                             template_result.warnings.begin(),
                             template_result.warnings.end());
        
        // Decrypt encrypted sections
        config = encryption_manager_->decrypt_sections(config);
        
        // Validate configuration
        for (auto& validator : validators_) {
            auto validation_result = validator->validate(config);
            if (!validation_result.is_valid) {
                throw std::runtime_error("Configuration validation failed: " + 
                                       validation_result.error_message);
            }
            result.warnings.insert(result.warnings.end(),
                                 validation_result.warnings.begin(),
                                 validation_result.warnings.end());
        }
        
        // Check for migrations
        if (version_manager_->needs_migration(config)) {
            auto migration_result = migration_engine_->migrate(config);
            config = migration_result.migrated_config;
            result.migrations_applied = migration_result.applied_migrations;
            result.requires_restart = migration_result.requires_restart;
        }
        
        result.config = config;
        return result;
    }
    
    void setup_config_watching(const std::string& config_path,
                             std::function<void(const nlohmann::json&)> callback) {
        std::string format = detect_format(config_path);
        auto watcher = watchers_[format]->create_watcher(config_path, callback);
        watcher->start_watching();
    }
};
```

**Problems**:
- Complex framework for simple configuration needs
- Multiple layers of abstraction obscure simple operations
- Template engine when environment variables suffice
- Version management complexity rarely needed

**✅ GOOD: Simple Configuration Loading**
```cpp
// Simple, direct configuration handling
class ConfigManager {
private:
    std::string config_path_;
    nlohmann::json current_config_;
    mutable std::shared_mutex config_mutex_;
    
public:
    ConfigManager(const std::string& config_path) : config_path_(config_path) {
        reload_config();
    }
    
    void reload_config() {
        std::ifstream file(config_path_);
        if (!file.is_open()) {
            throw std::runtime_error("Cannot open config file: " + config_path_);
        }
        
        std::string content((std::istreambuf_iterator<char>(file)),
                           std::istreambuf_iterator<char>());
        
        // Support environment variable substitution
        std::string expanded = expand_env_vars(content);
        
        // Parse JSON/YAML (assuming JSON for simplicity)
        nlohmann::json new_config = nlohmann::json::parse(expanded);
        
        // Simple validation
        validate_config(new_config);
        
        // Update config atomically
        {
            std::unique_lock<std::shared_mutex> lock(config_mutex_);
            current_config_ = std::move(new_config);
        }
    }
    
    nlohmann::json get_config() const {
        std::shared_lock<std::shared_mutex> lock(config_mutex_);
        return current_config_;
    }
    
    template<typename T>
    T get_value(const std::string& path, const T& default_value = T{}) const {
        std::shared_lock<std::shared_mutex> lock(config_mutex_);
        
        // Simple path traversal (e.g., "server.port")
        nlohmann::json current = current_config_;
        std::istringstream iss(path);
        std::string segment;
        
        while (std::getline(iss, segment, '.')) {
            if (current.contains(segment)) {
                current = current[segment];
            } else {
                return default_value;
            }
        }
        
        return current.get<T>();
    }
    
private:
    void validate_config(const nlohmann::json& config) {
        // Essential validations only
        if (!config.contains("server") || !config["server"].contains("port")) {
            throw std::runtime_error("Missing required server.port configuration");
        }
        
        if (config["server"]["port"].get<int>() <= 0) {
            throw std::runtime_error("Invalid port number");
        }
        
        if (!config.contains("ai_providers") || config["ai_providers"].empty()) {
            throw std::runtime_error("No AI providers configured");
        }
    }
    
    std::string expand_env_vars(const std::string& content) {
        std::string result = content;
        std::regex env_var_regex(R"(\$\{([^}]+)\})");
        std::smatch match;
        
        while (std::regex_search(result, match, env_var_regex)) {
            std::string var_name = match[1].str();
            const char* env_value = std::getenv(var_name.c_str());
            
            std::string replacement = env_value ? env_value : "";
            result.replace(match.position(), match.length(), replacement);
        }
        
        return result;
    }
};
```

### ❌ BAD: Complex State Management
**Problem**: Implementing complex state machines when simple states suffice

```cpp
// DON'T DO THIS - Over-engineered state management
enum class PipelineState {
    Initializing,
    ConfigurationLoading,
    ConfigurationValidating,
    ComponentsStarting,
    HealthChecking,
    Running,
    Degraded,
    Maintenance,
    ShuttingDown,
    Error,
    Stopped
};

enum class PipelineEvent {
    Initialize,
    ConfigLoaded,
    ConfigValid,
    ComponentsStarted,
    HealthCheckPassed,
    HealthCheckFailed,
    ErrorOccurred,
    MaintenanceRequested,
    ShutdownRequested,
    Recovery
};

class PipelineStateMachine {
private:
    PipelineState current_state_ = PipelineState::Stopped;
    std::vector<StateTransition> state_history_;
    std::unordered_map<std::pair<PipelineState, PipelineEvent>, PipelineState, 
                       boost::hash<std::pair<PipelineState, PipelineEvent>>> transition_rules_;
    std::unordered_map<PipelineState, std::unique_ptr<StateValidator>> state_validators_;
    std::unordered_map<std::pair<PipelineState, PipelineState>, std::unique_ptr<TransitionGuard>,
                       boost::hash<std::pair<PipelineState, PipelineState>>> transition_guards_;
    std::unordered_map<PipelineState, std::vector<std::unique_ptr<StateListener>>> state_listeners_;
    mutable std::mutex state_mutex_;
    
    struct StateTransition {
        PipelineState from;
        PipelineState to;
        PipelineEvent event;
        std::chrono::system_clock::time_point timestamp;
        std::string context;
    };
    
public:
    PipelineStateMachine() {
        initialize_transition_rules();
        initialize_state_validators();
        initialize_transition_guards();
    }
    
    void transition_to(PipelineState target_state, PipelineEvent event, const std::string& context = "") {
        std::lock_guard<std::mutex> lock(state_mutex_);
        
        // Check if transition is allowed
        auto transition_key = std::make_pair(current_state_, event);
        auto allowed_target_it = transition_rules_.find(transition_key);
        
        if (allowed_target_it == transition_rules_.end()) {
            throw std::runtime_error("Invalid transition from " + state_to_string(current_state_) +
                                    " with event " + event_to_string(event));
        }
        
        if (allowed_target_it->second != target_state) {
            throw std::runtime_error("Unexpected target state for transition");
        }
        
        // Run transition guard
        auto guard_key = std::make_pair(current_state_, target_state);
        auto guard_it = transition_guards_.find(guard_key);
        if (guard_it != transition_guards_.end()) {
            if (!guard_it->second->check_transition(current_state_, target_state)) {
                throw std::runtime_error("Transition guard failed");
            }
        }
        
        // Validate target state
        auto validator_it = state_validators_.find(target_state);
        if (validator_it != state_validators_.end()) {
            if (!validator_it->second->validate_state(target_state)) {
                throw std::runtime_error("Target state validation failed");
            }
        }
        
        // Record transition
        state_history_.push_back({
            current_state_,
            target_state,
            event,
            std::chrono::system_clock::now(),
            context
        });
        
        // Update state
        PipelineState old_state = current_state_;
        current_state_ = target_state;
        
        // Notify listeners
        auto listeners_it = state_listeners_.find(target_state);
        if (listeners_it != state_listeners_.end()) {
            for (auto& listener : listeners_it->second) {
                listener->on_state_entered(target_state, old_state);
            }
        }
    }
    
private:
    void initialize_transition_rules() {
        // Define all valid state transitions
        transition_rules_[{PipelineState::Stopped, PipelineEvent::Initialize}] = PipelineState::Initializing;
        transition_rules_[{PipelineState::Initializing, PipelineEvent::ConfigLoaded}] = PipelineState::ConfigurationLoading;
        // ... 50+ more transition rules
    }
};
```

**Problems**:
- Complex state machine for simple operational states
- Transition rules and guards add unnecessary complexity
- State history tracking rarely useful in practice
- Multiple listeners create callback complexity

**✅ GOOD: Simple State Tracking**
```cpp
// Simple state management that focuses on what matters
enum class PipelineStatus {
    Starting,
    Running,
    Error,
    Stopped
};

class PipelineManager {
private:
    std::atomic<PipelineStatus> status_{PipelineStatus::Stopped};
    std::string last_error_;
    mutable std::mutex error_mutex_;
    
public:
    void set_status(PipelineStatus status, const std::string& error_message = "") {
        status_.store(status);
        
        if (status == PipelineStatus::Error && !error_message.empty()) {
            std::lock_guard<std::mutex> lock(error_mutex_);
            last_error_ = error_message;
        }
    }
    
    PipelineStatus get_status() const {
        return status_.load();
    }
    
    bool is_healthy() const {
        return status_.load() == PipelineStatus::Running;
    }
    
    std::string get_last_error() const {
        std::lock_guard<std::mutex> lock(error_mutex_);
        return last_error_;
    }
    
    nlohmann::json get_status_json() const {
        std::string status_str;
        switch (status_.load()) {
            case PipelineStatus::Starting: status_str = "starting"; break;
            case PipelineStatus::Running: status_str = "running"; break;
            case PipelineStatus::Error: status_str = "error"; break;
            case PipelineStatus::Stopped: status_str = "stopped"; break;
        }
        
        nlohmann::json result = {
            {"status", status_str},
            {"timestamp", std::chrono::duration_cast<std::chrono::seconds>(
                std::chrono::system_clock::now().time_since_epoch()).count()}
        };
        
        if (status_.load() == PipelineStatus::Error) {
            std::lock_guard<std::mutex> lock(error_mutex_);
            result["error"] = last_error_;
        }
        
        return result;
    }
};
```

## 4. Performance Optimization Premature Complexity

### ❌ BAD: Over-Engineered Connection Pooling
**Problem**: Complex connection pooling when simple solutions work

```cpp
// DON'T DO THIS - Over-engineered connection pooling
template<typename ConnectionType>
class AdvancedConnectionPool {
private:
    struct PooledConnection {
        std::unique_ptr<ConnectionType> connection;
        std::chrono::steady_clock::time_point last_used;
        std::chrono::steady_clock::time_point created;
        uint32_t use_count = 0;
        bool is_healthy = true;
    };
    
    std::vector<PooledConnection> connections_;
    std::queue<std::unique_ptr<ConnectionRequest>> wait_queue_;
    std::unique_ptr<AutoScalingAlgorithm> scaling_algorithm_;
    std::unique_ptr<ConnectionFactory<ConnectionType>> connection_factory_;
    std::unique_ptr<LoadBalancer> load_balancer_;
    std::unique_ptr<HealthMonitor> health_monitor_;
    
    size_t min_connections_;
    size_t max_connections_;
    std::chrono::seconds idle_timeout_;
    std::chrono::seconds max_lifetime_;
    std::optional<std::string> validation_query_;
    
    mutable std::mutex pool_mutex_;
    std::condition_variable pool_cv_;
    
    struct ConnectionRequest {
        std::chrono::steady_clock::time_point timeout;
        std::promise<std::unique_ptr<ConnectionType>> response;
    };
    
public:
    std::future<std::unique_ptr<ConnectionType>> get_connection(std::chrono::seconds timeout) {
        std::unique_lock<std::mutex> lock(pool_mutex_);
        
        // Check for available healthy connections
        for (auto& pooled : connections_) {
            if (!pooled.connection || !pooled.is_healthy) continue;
            
            auto now = std::chrono::steady_clock::now();
            
            // Check if connection has expired
            if (now - pooled.created > max_lifetime_) {
                pooled.connection.reset();
                continue;
            }
            
            // Validate connection if needed
            if (validation_query_.has_value()) {
                if (!validate_connection(*pooled.connection, validation_query_.value())) {
                    pooled.is_healthy = false;
                    continue;
                }
            }
            
            // Update usage stats
            pooled.last_used = now;
            pooled.use_count++;
            
            auto connection = std::move(pooled.connection);
            pooled.connection = nullptr;
            
            auto promise = std::make_unique<std::promise<std::unique_ptr<ConnectionType>>>();
            auto future = promise->get_future();
            promise->set_value(std::move(connection));
            
            return future;
        }
        
        // Check if we can create new connection
        size_t active_connections = count_active_connections();
        if (active_connections < max_connections_) {
            return create_new_connection();
        }
        
        // Wait for available connection
        auto request = std::make_unique<ConnectionRequest>();
        request->timeout = std::chrono::steady_clock::now() + timeout;
        auto future = request->response.get_future();
        
        wait_queue_.push(std::move(request));
        
        // Trigger auto-scaling evaluation
        scaling_algorithm_->evaluate_scaling_need(active_connections, wait_queue_.size());
        
        return future;
    }
    
    void return_connection(std::unique_ptr<ConnectionType> connection) {
        std::lock_guard<std::mutex> lock(pool_mutex_);
        
        // Find empty slot or create new one
        for (auto& pooled : connections_) {
            if (!pooled.connection) {
                pooled.connection = std::move(connection);
                pooled.last_used = std::chrono::steady_clock::now();
                
                // Notify waiting requests
                if (!wait_queue_.empty()) {
                    auto request = std::move(wait_queue_.front());
                    wait_queue_.pop();
                    
                    if (std::chrono::steady_clock::now() < request->timeout) {
                        request->response.set_value(std::move(pooled.connection));
                        pooled.connection = nullptr;
                    }
                }
                
                return;
            }
        }
        
        // Pool is full, create new slot if under max_connections
        if (connections_.size() < max_connections_) {
            connections_.push_back({std::move(connection), 
                                  std::chrono::steady_clock::now(),
                                  std::chrono::steady_clock::now(), 1, true});
        }
        // Otherwise discard connection
    }
};
```

**Problems**:
- Complex scaling algorithms for predictable load patterns
- Over-abstracted connection management
- Health monitoring complexity for HTTP connections
- Wait queue management when simple retry works

**✅ GOOD: Simple Connection Management**
```cpp
// Simple connection management using standard HTTP client
#include <curl/curl.h>

class SimpleConnectionManager {
private:
    CURL* curl_handle_;
    std::chrono::seconds timeout_;
    uint32_t max_retries_;
    
public:
    SimpleConnectionManager(std::chrono::seconds timeout = std::chrono::seconds(30),
                          uint32_t max_retries = 3)
        : timeout_(timeout), max_retries_(max_retries) {
        
        curl_handle_ = curl_easy_init();
        if (!curl_handle_) {
            throw std::runtime_error("Failed to initialize CURL");
        }
        
        // Set common options
        curl_easy_setopt(curl_handle_, CURLOPT_TIMEOUT, timeout_.count());
        curl_easy_setopt(curl_handle_, CURLOPT_FOLLOWLOCATION, 1L);
        curl_easy_setopt(curl_handle_, CURLOPT_SSL_VERIFYPEER, 1L);
    }
    
    ~SimpleConnectionManager() {
        if (curl_handle_) {
            curl_easy_cleanup(curl_handle_);
        }
    }
    
    struct HttpResponse {
        std::string body;
        long status_code;
        std::string error_message;
    };
    
    HttpResponse send_request(const std::string& url, 
                            const std::string& data = "",
                            const std::vector<std::string>& headers = {}) {
        HttpResponse response;
        std::string error_buffer;
        error_buffer.resize(CURL_ERROR_SIZE);
        
        for (uint32_t attempt = 0; attempt <= max_retries_; ++attempt) {
            response.body.clear();
            response.error_message.clear();
            
            // Set URL
            curl_easy_setopt(curl_handle_, CURLOPT_URL, url.c_str());
            
            // Set data if provided
            if (!data.empty()) {
                curl_easy_setopt(curl_handle_, CURLOPT_POSTFIELDS, data.c_str());
            }
            
            // Set headers
            struct curl_slist* header_list = nullptr;
            for (const auto& header : headers) {
                header_list = curl_slist_append(header_list, header.c_str());
            }
            if (header_list) {
                curl_easy_setopt(curl_handle_, CURLOPT_HTTPHEADER, header_list);
            }
            
            // Set write callback
            curl_easy_setopt(curl_handle_, CURLOPT_WRITEFUNCTION, WriteCallback);
            curl_easy_setopt(curl_handle_, CURLOPT_WRITEDATA, &response.body);
            
            // Set error buffer
            curl_easy_setopt(curl_handle_, CURLOPT_ERRORBUFFER, error_buffer.data());
            
            // Perform request
            CURLcode res = curl_easy_perform(curl_handle_);
            
            // Clean up headers
            if (header_list) {
                curl_slist_free_all(header_list);
            }
            
            if (res == CURLE_OK) {
                curl_easy_getinfo(curl_handle_, CURLINFO_RESPONSE_CODE, &response.status_code);
                return response;
            } else {
                response.error_message = error_buffer;
                
                if (attempt < max_retries_) {
                    // Simple exponential backoff
                    auto delay = std::chrono::milliseconds(100 * (1 << attempt));
                    std::this_thread::sleep_for(delay);
                }
            }
        }
        
        response.status_code = 0;
        return response;
    }
    
private:
    static size_t WriteCallback(void* contents, size_t size, size_t nmemb, std::string* output) {
        size_t total_size = size * nmemb;
        output->append(static_cast<char*>(contents), total_size);
        return total_size;
    }
};
```

### ❌ BAD: Complex Caching Strategies
**Problem**: Over-engineering caching when simple TTL cache works

```cpp
// DON'T DO THIS - Over-engineered caching system
enum class CacheLevel { L1, L2, L3 };
enum class WriteStrategy { WriteThrough, WriteBack, WriteAround };
enum class EvictionPolicy { LRU, LFU, FIFO, Random };

template<typename K, typename V>
class AdvancedCacheManager {
private:
    std::unique_ptr<LRUCache<K, V>> l1_cache_;
    std::unique_ptr<DiskCache<K, V>> l2_cache_;
    std::unique_ptr<DistributedCache<K, V>> l3_cache_;
    std::unique_ptr<CacheCoherenceManager> cache_coherence_;
    std::unordered_map<CacheLevel, EvictionPolicy> eviction_policies_;
    std::unordered_map<CacheLevel, WriteStrategy> write_strategies_;
    std::unique_ptr<ConsistencyManager> consistency_manager_;
    std::unique_ptr<CacheWarmingManager> cache_warming_;
    
    mutable std::shared_mutex cache_mutex_;
    
public:
    std::optional<V> get(const K& key) {
        std::shared_lock<std::shared_mutex> lock(cache_mutex_);
        
        // Check L1 cache
        if (auto value = l1_cache_->get(key)) {
            update_access_pattern(key, CacheLevel::L1);
            return value;
        }
        
        // Check L2 cache
        if (auto value = l2_cache_->get(key)) {
            // Promote to L1
            l1_cache_->put(key, *value);
            update_access_pattern(key, CacheLevel::L2);
            return value;
        }
        
        // Check L3 cache
        if (auto value = l3_cache_->get(key)) {
            // Promote through cache levels
            l2_cache_->put(key, *value);
            l1_cache_->put(key, *value);
            update_access_pattern(key, CacheLevel::L3);
            return value;
        }
        
        return std::nullopt;
    }
    
    void put(const K& key, const V& value) {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        
        // Determine optimal cache level based on access patterns
        CacheLevel target_level = analyze_optimal_cache_level(key, value);
        
        // Apply write strategy
        WriteStrategy strategy = write_strategies_[target_level];
        
        switch (strategy) {
            case WriteStrategy::WriteThrough:
                write_to_all_levels(key, value);
                break;
            case WriteStrategy::WriteBack:
                write_to_level(key, value, target_level);
                mark_for_writeback(key, target_level);
                break;
            case WriteStrategy::WriteAround:
                write_to_backing_store(key, value);
                break;
        }
        
        // Update cache coherence
        cache_coherence_->invalidate_stale_copies(key);
    }
    
private:
    void update_access_pattern(const K& key, CacheLevel level) {
        // Complex access pattern analysis
        AccessPattern pattern;
        pattern.key = key;
        pattern.level = level;
        pattern.timestamp = std::chrono::steady_clock::now();
        pattern.frequency = calculate_access_frequency(key);
        pattern.recency = calculate_recency_score(key);
        
        access_patterns_[key].push_back(pattern);
        
        // Cleanup old patterns
        cleanup_old_patterns(key);
    }
    
    CacheLevel analyze_optimal_cache_level(const K& key, const V& value) {
        auto patterns = access_patterns_[key];
        
        if (patterns.empty()) {
            return CacheLevel::L1; // Default for new items
        }
        
        // Complex analysis based on access patterns, value size, etc.
        double l1_score = calculate_l1_score(patterns, value);
        double l2_score = calculate_l2_score(patterns, value);
        double l3_score = calculate_l3_score(patterns, value);
        
        if (l1_score > l2_score && l1_score > l3_score) return CacheLevel::L1;
        if (l2_score > l3_score) return CacheLevel::L2;
        return CacheLevel::L3;
    }
};
```

**Problems**:
- Multiple cache levels add complexity without clear benefit
- Cache coherence complexity for single-node deployments
- Write strategies over-engineered for simple use cases
- Access pattern analysis overhead for simple caching needs

**✅ GOOD: Simple TTL Cache**
```cpp
// Simple TTL cache that solves 90% of caching needs
template<typename K, typename V>
class SimpleCache {
private:
    struct CacheEntry {
        V value;
        std::chrono::steady_clock::time_point expires_at;
    };
    
    std::unordered_map<K, CacheEntry> entries_;
    mutable std::shared_mutex cache_mutex_;
    std::chrono::seconds default_ttl_;
    size_t max_size_;
    
public:
    SimpleCache(std::chrono::seconds default_ttl = std::chrono::seconds(300),
               size_t max_size = 1000)
        : default_ttl_(default_ttl), max_size_(max_size) {}
    
    std::optional<V> get(const K& key) {
        std::shared_lock<std::shared_mutex> lock(cache_mutex_);
        
        auto it = entries_.find(key);
        if (it != entries_.end()) {
            if (it->second.expires_at > std::chrono::steady_clock::now()) {
                return it->second.value;
            }
        }
        
        return std::nullopt;
    }
    
    void put(const K& key, const V& value, 
             std::optional<std::chrono::seconds> ttl = std::nullopt) {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        
        auto expires_at = std::chrono::steady_clock::now() + 
                         ttl.value_or(default_ttl_);
        
        // Simple eviction: remove expired entries if cache is full
        if (entries_.size() >= max_size_) {
            cleanup_expired_entries();
            
            // If still full, remove oldest entries (simple FIFO)
            if (entries_.size() >= max_size_) {
                auto oldest_it = std::min_element(entries_.begin(), entries_.end(),
                    [](const auto& a, const auto& b) {
                        return a.second.expires_at < b.second.expires_at;
                    });
                entries_.erase(oldest_it);
            }
        }
        
        entries_[key] = {value, expires_at};
    }
    
    void remove(const K& key) {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        entries_.erase(key);
    }
    
    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(cache_mutex_);
        return entries_.size();
    }
    
    void clear() {
        std::unique_lock<std::shared_mutex> lock(cache_mutex_);
        entries_.clear();
    }
    
private:
    void cleanup_expired_entries() {
        auto now = std::chrono::steady_clock::now();
        
        for (auto it = entries_.begin(); it != entries_.end();) {
            if (it->second.expires_at <= now) {
                it = entries_.erase(it);
            } else {
                ++it;
            }
        }
    }
};

// Usage example:
/*
SimpleCache<std::string, nlohmann::json> response_cache;

// Cache AI provider response for 5 minutes
response_cache.put("openai_request_123", response_json, std::chrono::minutes(5));

// Try to get cached response
if (auto cached = response_cache.get("openai_request_123")) {
    return *cached; // Use cached response
}
*/
```

## Prevention Strategies for Colloquium

### 1. Complexity Budget
- **Start Simple**: Always implement the simplest solution first
- **Complexity Justification**: Require clear justification for any added complexity
- **Complexity Debt**: Track and pay down complexity debt regularly
- **Complexity Review**: Regular architecture reviews to identify over-engineering

### 2. Performance First, Optimization Later
- **Measure Before Optimizing**: Always profile before adding performance complexity
- **Simple Solutions First**: Try simple solutions before complex optimizations
- **Benchmarking**: Establish clear benchmarks and only optimize when they're not met
- **Real-World Testing**: Test with realistic workloads, not synthetic benchmarks

### 3. Operational Simplicity
- **Single Binary**: Maintain single binary deployment model
- **Simple Configuration**: Prefer YAML over complex DSLs
- **Clear Error Messages**: Focus on debuggability over sophisticated error handling
- **Documentation Over Automation**: Sometimes clear documentation is better than complex automation

### 4. Community Feedback Loops
- **Early Feedback**: Get feedback from real users early and often
- **Feature Usage**: Track which features are actually used
- **Complexity Complaints**: Listen when users say things are too complex
- **Success Metrics**: Measure time-to-first-success for new users

### 5. Engineering Discipline
- **Code Reviews**: Focus on simplicity in code reviews
- **Refactoring**: Regular refactoring to remove unnecessary complexity
- **Testing**: Comprehensive testing prevents over-engineering for edge cases
- **Documentation**: Clear documentation prevents feature creep

## Success Criteria for Avoiding Anti-Patterns

### Development Metrics
- **Time to First Pipeline**: New developers can create working pipeline in < 30 minutes
- **Configuration Complexity**: Pipeline configurations are < 50 lines of YAML
- **Deployment Simplicity**: Single command deployment with clear error messages
- **Debug Efficiency**: Most issues can be debugged with logs and simple metrics

### Operational Metrics
- **Memory Usage**: Stable memory usage under load (no complex caching or pooling)
- **CPU Efficiency**: Linear relationship between load and CPU usage
- **Error Recovery**: Automatic recovery from 90%+ of issues without operator intervention
- **Monitoring Clarity**: Critical issues visible in < 5 core metrics

### Code Quality Metrics
- **Function Length**: No function longer than 50 lines
- **File Organization**: Related functionality grouped, < 500 lines per file
- **Test Coverage**: > 90% test coverage with simple, readable tests
- **Documentation**: Every public API documented with examples

These anti-patterns represent real traps that lead to over-engineered systems that are difficult to deploy, debug, and maintain. By following the simple alternatives and prevention strategies, Colloquium can maintain the operational simplicity and reliability that made Permuto and Computo successful while adding the stream processing capabilities needed for AI workflows.