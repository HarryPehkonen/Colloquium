# EXECUTION_GUIDANCE.md

## Introduction: Guiding Principles for Implementation

The Colloquium project is defined by a comprehensive set of documents: `VISION`, `REQUIREMENTS`, `ARCHITECTURE`, `ANTI_PATTERNS`, and `TECHNICAL_IMPLEMENTATION`. These documents describe *what* to build and *how* to structure it.

This document serves a different purpose: it provides high-level guidance on **how to think about the implementation**. It is intended to be a touchstone for the development team to ensure that the spirit of the design—balancing power with simplicity—is preserved in every line of code. These principles should guide decision-making, code reviews, and architectural discussions throughout the project's lifecycle.

## 1. The Simplicity Paradox

**The Principle:** Colloquium's primary value proposition is "operational simplicity" combined with enterprise-grade capabilities. A single developer should be able to deploy and manage it effectively.

**The Challenge:** The required feature set (e.g., intelligent routing, circuit breakers, cost optimization) is inherently complex. There is a significant risk that this internal complexity will "leak" into the user's configuration, the operational experience, or the cognitive load required to use the system.

**Guiding Questions for Development:**
- Is this the absolute simplest way to deliver the required functionality? Can we provide 80% of the value with 20% of the complexity?
- Does this feature add cognitive load to the user's YAML configuration? Could this be handled by a sensible default or convention over configuration?
- How would a single developer operate, debug, and recover this feature at 3 AM during an outage? (The "3 AM Test").
- Are we building something "clever" when something "boring and obvious" would suffice?

**Actionable Recommendations:**
- **Adhere to `ANTI_PATTERNS.md`:** Treat the anti-patterns document as a sacred text. Every new component or complex feature should be reviewed against it.
- **Implement in Layers:** Start with the simplest possible version of a feature. Add advanced capabilities (like auto-scaling or complex heuristics) only after the simple version is proven and real-world needs justify the added complexity.
- **Prioritize Convention Over Configuration:** Strive to make the system work well out-of-the-box with minimal configuration. Only expose knobs and dials when absolutely necessary.

## 2. The Performance-Complexity Trade-off

**The Principle:** The architecture targets elite performance using modern C++ techniques (zero-copy, lock-free algorithms, SIMD).

**The Challenge:** These advanced techniques are notoriously difficult to implement correctly and can introduce subtle, non-deterministic bugs. A poorly implemented lock-free algorithm is infinitely worse than a simple, well-understood mutex. Premature optimization can lead to unmaintainable code for negligible real-world gains.

**Guiding Questions for Development:**
- **Have we profiled this code?** Is there clear, unambiguous data showing this specific section is a performance bottleneck?
- What is the simplest, safest implementation using standard library components? Is its performance "good enough" for the current phase?
- Does the team have the deep expertise required to correctly implement, test, and maintain this complex solution?
- What is our testing strategy for this specific piece of complex concurrent or low-level code? How can we prove its correctness?

**Actionable Recommendations:**
- **Measure, Don't Guess:** No performance optimization should be permitted without profiling data to justify it.
- **Simple and Correct First:** Always start with the simplest, most correct, and most readable implementation. Only replace it with a more complex, high-performance version when benchmarks prove it is necessary.
- **Isolate Complexity:** Confine the most complex performance-critical code to small, well-defined, and intensely tested components.

## 3. The Computo Integration Bottleneck

**The Principle:** Colloquium leverages the `computo` library as the core transformation engine, which is a foundational architectural assumption.

**The Challenge:** The entire system's throughput, stability, and security are capped by the `computo` engine's capabilities. Any performance limitations, memory leaks, or concurrency bugs within `computo` will become a critical, system-wide failure point.

**Guiding Questions for Development:**
- What is the measured, worst-case performance of `computo::execute` under heavy, multi-threaded load?
- What is our strategy for sandboxing `computo` execution? What happens if a user script has an infinite loop, allocates excessive memory, or throws an unexpected exception?
- Have we defined a clear "performance budget" (e.g., in microseconds) for a typical transformation?

**Actionable Recommendations:**
- **Priority Zero - Validate the Core Assumption:** Before significant development on the Colloquium shell, the very first task should be to create a standalone, multi-threaded benchmark harness for the `computo` library. This will stress-test its performance and concurrency characteristics to validate that it can meet the project's throughput targets.
- **Develop a Sandboxing Strategy:** Implement robust mechanisms to isolate `computo` execution, including strict timeouts and memory limits, to prevent a single bad script from impacting the entire stream processor.

## 4. The Foundation: Build System and Dependency Management

**The Principle:** Colloquium is a professional, cross-platform C++ application intended for production deployment.

**The Challenge:** C++ lacks a universally adopted, built-in package manager, making dependency management and the creation of a consistent, cross-platform build environment a significant undertaking. This is often an afterthought, which can cripple a project's velocity.

**Guiding Questions for Development:**
- How does a new developer clone the repository and get a successful build and passing tests in under 30 minutes?
- How do we manage the versions of our third-party dependencies (e.g., `nlohmann/json`, `simdjson`, `gtest`) to ensure reproducible builds?
- How does our build system support compilation and testing on our target platforms (Linux, macOS, Windows)?

**Actionable Recommendations:**
- **Establish the Foundation First:** From Day 1, select and configure the project's build system (e.g., CMake) and a dependency manager (e.g., Conan or vcpkg).
- **Automate Immediately:** The CI/CD pipeline for building and testing the project should be one of the first things created.
- **Document the Process:** The developer onboarding process, including system setup, dependencies, and build commands, must be clearly documented and maintained. The build system is a first-class citizen of the architecture.

## Conclusion

This guidance is not intended to be restrictive. It is designed to empower the team to navigate the inherent complexities of building a high-performance distributed system. By regularly consulting these principles, we can ensure that we remain true to the project's vision, avoid common pitfalls, and ultimately deliver a product that is as simple to operate as it is powerful to use.
