## Key Problems with the Current Auto Quote Process

### 1. Tight Coupling Between Auto Quote and Guidewire
The Auto Quote process is heavily dependent on Guidewire throughout the quote journey rather than using Guidewire primarily as the system of record at policy creation/bind.

This means changes in Guidewire product configuration, APIs, workflows, or data structures can directly impact the digital quote experience.

### 2. Tight Coupling Between Guidewire and Earnix
Guidewire and Earnix are closely integrated during the quote process, creating a strong runtime dependency between two major SaaS platforms.

As a result, the quote process depends on the availability, performance, and behavior of both platforms even when their capabilities are not required for every step of the customer journey.

### 3. Excessive Runtime Dependencies
The current quote journey requires multiple downstream systems to participate synchronously.

A failure, timeout, latency issue, or maintenance event in Guidewire or Earnix can potentially impact the customer's ability to continue or complete a quote.

### 4. Poor Separation of Responsibilities
The current design mixes several responsibilities across the Auto Quote application, Guidewire, and Earnix, including:

- Customer and driver data collection
- Vehicle and address information
- Product/rule validation
- Quote orchestration
- Rating
- Policy-related processing
- Product configuration

This makes ownership unclear and increases the complexity of the overall solution.

### 5. Guidewire is Being Used for More Than Its Core Responsibility
Guidewire is participating in the quote journey before the customer has actually decided to bind the policy.

This creates unnecessary dependency on the policy administration platform for what is primarily a digital acquisition and quote-creation process.

### 6. Limited Ability to Independently Evolve the Digital Quote Experience
Because the quote journey is closely tied to Guidewire and Earnix, changes to the digital experience may require corresponding changes or coordination with downstream platforms.

This reduces team autonomy and increases the lead time and risk associated with enhancements.

### 7. Increased Failure Propagation
A downstream failure can propagate upstream into the customer-facing quote experience.

For example:

**Auto Quote → Guidewire → Earnix**

If Earnix is unavailable or slow, the impact can propagate through Guidewire and ultimately affect the Auto Quote experience.

This creates a larger failure domain than necessary.

### 8. Performance and Scalability Concerns
The customer-facing application is dependent on the response time and scalability characteristics of Guidewire and Earnix.

As quote volume increases, scaling the digital channel independently becomes more difficult because downstream systems remain part of the synchronous transaction path.

### 9. Difficult Resiliency and Availability Management
The current architecture makes it difficult to implement effective resiliency patterns because the quote process is tightly coupled to external platforms.

Timeouts, retries, circuit breakers, bulkheads, fallback strategies, and graceful degradation become more complicated when business processing spans multiple tightly coupled systems.

### 10. Complex Change Management
A relatively small business change can require changes across multiple components because business rules, product logic, rating, and quote orchestration are distributed across Auto Quote, Guidewire, and Earnix.

This increases:

**Development effort → Testing effort → Release coordination → Production risk**

### 11. Difficult Testing
End-to-end testing requires multiple external systems to be available and correctly configured.

Testing a change in the Auto Quote journey may therefore require Guidewire and Earnix environments, test data, product configuration, and integration coordination.

### 12. Vendor Lock-in Within the Customer Journey
The current architecture places Guidewire and Earnix directly in the critical path of the digital customer experience.

This makes it harder to introduce new rating engines, replace components, change product services, or evolve the digital architecture independently in the future.

### 13. Business Logic is Distributed Across Multiple Platforms
Product validation, eligibility, rules, and rating-related logic may be distributed across Auto Quote, Guidewire, and Earnix.

This can result in duplicated rules, inconsistent behavior, and difficulty determining which system is the authoritative source for a particular business rule.

### 14. Difficult to Support a Fast Digital Experience
The customer is completing a multi-step quote journey, but the current architecture potentially requires repeated communication with backend SaaS platforms throughout that journey.

This creates unnecessary latency and makes it harder to provide a fast, responsive digital experience.

### 15. Architectural Boundary Between Quote and Policy Creation is Blurred
The quote process and policy administration process are closely intertwined.

The architecture would be simpler if the responsibilities were clearly separated:

**Quote Journey → Product/Rule Validation → Rating → Quote Persistence → Customer Acceptance → Policy Creation in Guidewire**

This would allow Guidewire to focus on policy administration while the digital quote domain owns the customer-facing quote experience.