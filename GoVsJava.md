For a medium-complexity microservices application (e.g., typical RESTful services with database interactions, message queues, etc., involving 10-20 services), running on AWS EKS, a Go implementation typically achieves significant infrastructure cost savings compared to a traditional Java implementation (e.g., Spring Boot). The savings primarily come from lower resource consumption and higher deployment density.
Main Sources of Savings

Memory Usage: Go binaries are lightweight with no JVM overhead—idle memory is typically 20-50MB, and under load usually 100-200MB. Java services often use 300-400MB idle and can reach 800MB+ under load. This allows Go pods to have much lower memory requests/limits.
CPU Efficiency: Go's goroutine concurrency model is lightweight, often resulting in better CPU utilization for the same workload.
Pod Density: On the same EC2 nodes, Go instances can achieve 2-3x higher density (e.g., 8 Go instances vs. 3 Java instances on the same node).
Image Size & Startup: Go container images are small (10-20MB vs. Java's 150-200MB), and cold starts are fast (<1s vs. 8-10s for Java). This improves autoscaling and rolling updates, indirectly reducing peak resource needs.
EKS Cost Breakdown: The EKS control plane fee is fixed (~$73/month per cluster) and identical for both, so it doesn't affect the comparison. Savings mainly come from worker nodes (EC2 instances or Fargate), which typically account for 80%+ of total cost.

Typical Savings Range
Based on multiple real-world cases and benchmarks (including production migrations):

50-70% savings on compute costs is common.
One production comparison (same traffic load): Java required 6 t3.medium instances (~$250/month), Go needed only 3 t3.small instances (~$75/month) → ~70% savings.
Another migration: After switching to Go, instance count halved and instance sizes reduced, resulting in 2.5x lower overall cost (~60% savings).
High-traffic scenario (7-10k RPS): Java needed 6 xlarge instances; Go needed 3 small instances → 50-60% savings.
From a memory perspective: Each Go pod saves 400-500MB vs. Java; with dozens of pods, this can translate to $1,000+ annual savings (at current AWS pricing).

Conservative estimate: For medium-complexity apps (without extreme Java optimizations), Go typically saves 40-60% on overall EKS infrastructure costs (primarily EC2/Fargate). On Fargate (billed by vCPU + GB-hour), the savings are even more directly tied to reduced resource allocation.

Influencing Factors

Application Characteristics: CPU/IO-intensive services see larger Go advantages; heavy business logic or complex Java library dependencies can narrow the gap.
Java Optimizations: Modern frameworks like Quarkus or GraalVM can bring Java memory/startup closer to Go levels, reducing the savings gap to 20-40%.
Scale: Small deployments have limited absolute savings (tens of dollars/month); medium-scale deployments ($1,000+/month infrastructure) show more noticeable benefits.
Other Costs: Go's faster deploys and smaller images can also reduce storage/traffic costs, though migration/development effort should be evaluated separately.

Overall, for medium-complexity microservices, Go offers substantial cost advantages on EKS—many teams report 40-70% savings after migration. If you have specific workload details (e.g., QPS, current Java resource requests), a more precise estimate is possible.
