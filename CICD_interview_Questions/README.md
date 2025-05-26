# Interview Questions


### _CICD Questions_
**1. How will the application code be tested and put into the production environment? what tests will exist?**
   **what ratio will there be between the different types of tests? what other types of static analysis should be in place?**
   * **Code Testing and Deployment Process**
      * Local Development
        * Developers write code and run unit tests locally.
        * Code is pushed to a feature branch in a version control system (e.g., GitHub/GitLab).
        * Static Analysis Tools
           * Code Quality & Style: ESLint, Checkstyle (Java)
           * Code coverage: JaCoCo (Java)
           * Code quality: SonarQube
      * Continuous Integration (CI) Pipeline
         * On every pull/merge request:
           * Build the application (fail early if it doesn't compile). 
           * Run automated test suites (unit, integration, end-to-end). 
           * Perform static code analysis (linters, security scans). 
           * Optionally run performance regression tests
      * Staging Environment
            * Deploy the application to a staging environment.
            * Run additional tests (e.g., load tests, smoke test, manual QA, end-to-end test).
      * Production Deployment
         * Deployment through CI/CD pipelines using tools like GitHub Actions. 
         * Deploy with canary releases, blue-green deployments, or rolling updates. 
         * Monitoring and alerting systems validate production health post-deploy.
**2. How should a pipeline be designed?** 
  * Designing a CI/CD pipeline for microservices should follow key principles: modularity, scalability, speed, safety, and observability
     * **Pipeline Architecture Overview**
        * Per Microservice Pipeline: Each service should have its own pipeline:
          * Owned and maintained by the team responsible for the service. 
          * Deployed independently. 
          * Versioned separately
        * **Pipeline Stages:** 
          * Source Code Management: 
            * Triggered by code changes (e.g., Git push).
            * Pull request validation.
            * Code review and approval process.
            * Static code analysis and linting/Unit tests execution/Integration tests execution in parallel.
            * Dependencies and images are scanned for vulnerabilities.
            * Run contract tests for API compatibility between services.
            * Build artifacts creation (e.g., Docker images).
            * Artifact storage (e.g., Docker registry).
            * Deployment to staging environment.
            * Acceptance tests execution (e.g., end-to-end tests).
     * **Design Best Practices**
        * Testing
            * Run unit tests first. 
            * Run integration tests with a test DB or via Testcontainers. 
            * Run contract tests for API compatibility between services. 
            * E2E tests only on staging or pre-prod.
        * Security : blackduck 
            * Scan code for vulnerabilities
            * Scan dependencies for known vulnerabilities.
        * Build & Artifact
            * The same artifact is promoted across environments (dev → staging → production) 
            * Create a Docker image per microservice tagged with Git SHA. 
            * Push images to a secure container registry (e.g., ECR, GCR, GHCR). 
            * Use immutable tags + proper cleanup of unused images
        * Deployments
            * Use kubernetes manifests or Helm charts for deployment.
            * Support canary or blue/green deployment strategies or rolling updates. 
            * Promote staging images to production only after validation.
     * Observability Built-In
        * All deployments should be traceable:
            * Which commit/branch deployed? 
            * Who triggered it? 
            * Which image/version?
        * Add:
            * Monitoring: Prometheus, Grafana 
            * Logging: Loki, ELK 
            * Tracing: Prometheus
            * Alerting: Grafana Loki 
          
**3. what are the ways to reduce deployment risk?** 
  * **Blue-Green Deployment**
    * Strategy: Maintain two environments (Blue and Green). Only one receives live traffic. 
    * Deploy to the inactive environment. 
    * Run smoke/E2E tests. 
    * Switch traffic if everything looks good. 
    * Rollback is easy: revert traffic to the previous environment.
  * **Canary Deployment**
    * Strategy: Deploy the new version to a small subset of users. 
    * Monitor performance and errors. 
    * Gradually increase traffic to the new version if no issues arise. 
    * Rollback is easy: redirect traffic back to the previous version.
  * **Feature Flags / Toggles**
    * Strategy: Deploy code with features disabled. Enable them incrementally via config/UI. 
    * Control exposure per user/region/team. 
    * Allow for fast rollback without redeploying. 
    * Use tools like LaunchDarkly, Unleash.
  * **Immutable Artifacts + GitOps**
    * Build once → tag image → deploy exact image everywhere. 
    * Use GitOps tools (e.g., ArgoCD, Flux) to manage deployments via Git.
  * **Health Checks & Circuit Breakers**
    * Liveness and readiness probes in Kubernetes. 
    * Deploys proceed only if services are healthy. 
    * Add circuit breakers (e.g., in Istio, Spring Cloud) to isolate failures.
  * **monitoring tools & Observability: Logs, Metrics, Traces**
    * Monitoring: Prometheus, Grafana 
    * Logging: Loki, ELK 
    * Tracing: Prometheus
    * Alerting: Grafana Loki 

### _Monitoring and Application health Questions_
**1. How can we ensure that the application operates correctly and works as intended?** 
    * **Ensuring Application Correctness**
      * **Automated Testing**
         * Unit Tests: Validate individual components.
         * Integration Tests: Ensure components work together.
         * End-to-End Tests: Simulate user scenarios to validate the entire flow.
         * Contract Tests: Verify API contracts between services.
      * **Code Reviews**
         * Peer reviews of code changes to catch issues early.
         * Use pull requests with mandatory reviews before merging.
      * **Static Code Analysis**
         * Use tools like SonarQube, ESLint, or Checkstyle to enforce coding standards and detect potential bugs.
      * **Continuous Integration (CI)**
         * Run tests automatically on every code change in a CI pipeline (e.g., GitHub Actions, Jenkins). 
      * 
1. How will the application be monitored? what metrics will be collected? how will the application health be determined?
   * **Application Monitoring**
     * **Metrics Collection**
       * Use Prometheus to collect application metrics.
       * Key Metrics:
         * Request Latency: Measure time taken to process requests.
         * Error Rates: Track 4xx and 5xx HTTP status codes.
         * Throughput: Count of requests per second.
         * Resource Utilization: CPU, memory, disk I/O, network usage.
         * Custom Application Metrics: Business-specific metrics (e.g., user sign-ups, transactions).
     * **Health Checks**
       * Implement liveness and readiness probes in Kubernetes.
       * Liveness Probe: Checks if the application is running. If it fails, Kubernetes restarts the pod.
       * Readiness Probe: Checks if the application is ready to serve traffic. If it fails, Kubernetes stops sending traffic to the pod.
     * **Alerting**
       * Set up alerting rules in Prometheus or Grafana based on thresholds for key metrics.
       * Use Alert manager to route alerts to appropriate channels (e.g., Slack, email).
     * **Logging**
       * Use a centralized logging solution (e.g., ELK stack, Loki) to collect and analyze logs.
       * Include structured logging with context (e.g., request ID, user ID) for better traceability.
2. how can we capture user frustration or usability issues (A/B testing, user interviews)? 
    * **Capturing User Frustration and Usability Issues**
      * **User Feedback**
         * In-app feedback forms or surveys to collect user opinions.
         * Net Promoter Score (NPS) surveys to gauge user satisfaction.
      * **A/B Testing** tools: LaunchDarkly
         * Implement A/B tests to compare different versions of features or UI elements.
         * Use tools like Google Optimize, Optimizely, or custom implementations.
         * Analyze metrics like conversion rates, engagement, and user retention.
      * **User Interviews and Usability Testing**
         * Conduct interviews with users to gather qualitative feedback.
         * Perform usability testing sessions to observe users interacting with the application.
         * Identify pain points and areas for improvement based on user behavior.
      * **Session Replay Tools**
         * Use tools like Hotjar or FullStory to record user sessions and analyze interactions.
         * Identify common navigation paths, clicks, and areas where users struggle.
      * **Analytics Tools**
         * Implement analytics tools (e.g., Google Analytics, Mixpanel) to track user behavior and engagement metrics.
         * Monitor key events (e.g., sign-ups, purchases) and funnel analysis to identify drop-off points.
**4. If an issue arises in production, how would you handle it? Explained about setting up alerts?**
   * When an issue arises in production, the goal is to detect it early, respond quickly, and resolve it with minimal user impact.
     * Handling Production Issues
        * **Detection**
            * Automated alerts from monitoring systems
            * User reports or support tickets
            * Error rate spikes in logging/tracing tools
        * Determine scope and severity:
            * Is it user-facing or internal?
            * How many users are affected?
            * Is it a performance degradation or total outage?
            * Label the issue (P1/P2/etc.) based on business impact.
        * Initial Response
            * Acknowledge alerts and notify team/on-call
            * Rollback if recent deploy caused it (canary/blue-green)
            * Mitigate impact if possible (e.g., feature flag off, rate limiting)
        * Root Cause Analysis (RCA)
            * Use logs, traces, metrics, dashboards to isolate the failure
            * Identify the root cause (e.g., code bug, infrastructure issue)
            * Collaborate with relevant teams (e.g., DevOps, SRE) if needed
            * Document findings in an incident report
        * Resolution
            * Fix the root cause in code or infrastructure
            * Test the fix in staging before deploying to production
            * Deploy the fix with proper monitoring in place
            * Communicate with stakeholders (e.g., status page update, incident report)
        * Postmortem
            * Document:
                * What happened 
                * What went wrong 
                * How it was fixed 
                * How to prevent recurrence 
   *  Setting Up Alerts 
       * Set Alerts on Key SLOs (Service-Level Objectives)
            * Error Rate: HTTP 5xx > 2% for 5 mins 
            * Availability: Request success rate < 99% in a rolling window 
            * Traffic Drops: 50% fewer requests than normal baseline
       * Infrastructure & System-Level Alerts
            * CPU/memory usage > 90% for 10 min 
            * Disk space low (< 10%)
            * Kubernetes pod restarts or crash loops 
            * Database connection pool saturation 
            * Queue or cache depth increasing abnormally 
       * Error & Log-Based Alerts
            * High volume of:
              * Exceptions (stack traces)
              * Timeouts 
              * Failed API calls
**5. What measures can be taken to minimize production issues**
   * Minimizing production issues is all about proactive quality, resilience, and early detection. Below are the most effective technical and organizational measures you can implement to reduce the frequency, severity, and duration of production problems:
        * Early and Automated Testing:
            * Unit, Integration, Contract, E2E tests in CI 
            * API contract testing to avoid breaking consumers
            * Test in staging with production-like data and traffic
        * Static & Dynamic Analysis:
            * Linting, type checks, static security scans
        * Secure and Hardened Builds
            * Build artifact once (e.g., Docker image) and promote through all environments  
            * Vulnerability Scanning: Scan dependencies and container images on every push
        * Reliable and Safe Deployments
            * Deployment Strategies:
                * Canary deployments: Gradually roll out to a small % of users 
                * Blue/Green deployments: Switch traffic after full validation 
                * Feature flags: Deploy code with flags off; enable gradually 
            * Rollback Mechanisms:
                * Automated or manual rollback support 
                * Keep previous stable versions deployable
            * Strong Observability
                * Metrics, Logs, and Traces:
                    * Use tools like Prometheus, Grafana, Loki, Jaeger 
                    * Track latency, error rate, resource usage, and custom KPIs
            * Infrastructure Resilience
                * Architecture Practices:
                    * Circuit breakers, retries with backoff, timeouts 
                    * Bulkheads to isolate failures 
                    * Graceful degradation (serve cached data, fallback UI)
                * Chaos Engineering:
                    * Simulate failures (e.g., latency, node loss) in non-prod environments 
                    * Tools: Chaos Mesh, Gremlin, Litmus 
            * Pre-Production Environments
                * Environments that Mirror Production:
                    * Use real service versions and data shape 
                    * Run load tests, smoke tests, and synthetic monitoring before release
**6.How would you scale the application? Explained about being proactive, monitoring traffic, having autoscaling in place, and setting up alerts for memory and CPU utilization?**
   * Scaling an application involves both proactive planning and reactive measures to handle an increased load. Here’s how you can effectively scale your application:
     * **Proactive Scaling**
        * **Capacity Planning**
            * Analyze historical traffic patterns to predict a future load.
            * Estimate peak traffic times and plan resources accordingly.
        * **Horizontal Scaling**
            * Design services to be stateless where possible, allowing easy scaling by adding more instances.
            * Use container orchestration (e.g., Kubernetes) to manage scaling of microservices.
        * **Load Balancing**
            * Implement load balancers (e.g., Nginx, HAProxy) to distribute traffic evenly across instances.
            * Use DNS-based load balancing for global distribution.
     * **Monitoring Traffic and Performance**
        * **Real-Time Monitoring**
            * Use tools like Prometheus, Grafana, or Datadog to monitor traffic, latency, error rates, and resource utilization.
            * Set up dashboards to visualize key metrics in real-time.
        * **Traffic Analysis**
            * Analyze incoming request patterns to identify bottlenecks or hotspots.
            * Use APM tools (e.g., New Relic, Dynatrace) for deep insights into application performance.
     * **Autoscaling**
        * **Dynamic Autoscaling**
            * Configure autoscaling policies based on CPU/memory utilization or custom metrics (e.g., request count).
            * Use Kubernetes Horizontal Pod Autoscaler (HPA) or AWS Auto Scaling Groups for automatic scaling.
        * **Predictive Autoscaling**
            * Implement predictive scaling based on historical data and machine learning models to anticipate traffic spikes.
     * **Setting Up Alerts**
        * **Resource Utilization Alerts**
            * Set alerts for high CPU (>80%) and memory (>80%) usage over a defined period (e.g., 5 minutes).
            * Alert on low disk space (<10% free) or high network latency (>200ms).
        * **Traffic Anomalies**
            * Alert on sudden spikes in traffic or error rates (e.g., 5xx errors > 1% of total requests).
        * **Alert Management**
            * Use alerting tools like Prometheus Alertmanager, PagerDuty, or Opsgenie to manage alerts and notify the right teams.
 
              




