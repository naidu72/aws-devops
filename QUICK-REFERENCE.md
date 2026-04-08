# DevOps Interview Quick Reference Guide

## Print This Out or Screenshot for Quick Review

---

## 🔥 Top 30 Interview Questions You MUST Know

### AWS Services (10 Questions)

1. **Explain IAM Roles vs IAM Users**
   - Users: Long-term credentials, for people/apps
   - Roles: Temporary credentials via STS, for services
   - Use roles for EC2, Lambda, cross-account

2. **Difference between ALB and NLB**
   - ALB: Layer 7 (application), path/host-based routing, microservices
   - NLB: Layer 4 (transport), extreme performance, 1M+ req/sec

3. **S3 Bucket Policy vs IAM Policy**
   - Bucket: Resource-based, attached to S3 bucket
   - IAM: Identity-based, attached to user/role
   - Both evaluated (intersection = final permission)

4. **RDS Multi-AZ vs Read Replicas**
   - Multi-AZ: Sync failover, same region, 0 data loss
   - Replicas: Async, any region, possible data loss

5. **When to use DynamoDB vs RDS?**
   - DynamoDB: Unstructured, massive scale, NoSQL
   - RDS: Structured, ACID, relationships needed

6. **How VPC Security Groups work**
   - Stateful firewall (instance-level)
   - Default deny inbound, allow outbound
   - Return traffic automatically allowed
   - Can reference other security groups

7. **What is NAT Gateway used for?**
   - Allows private instances to initiate outbound to internet
   - Returns responses allowed automatically
   - Requires Elastic IP
   - Goes in public subnet

8. **S3 Strong Read-After-Write Consistency**
   - Can read immediately after writing
   - No eventual consistency delay
   - Simplifies application logic
   - True for all S3 APIs (PutObject, DeleteObject, etc.)

9. **Lambda cold starts - when and why?**
   - When: First invocation, new function, new region
   - Why: Container initialization overhead
   - Mitigation: Provisioned concurrency, larger memory

10. **CloudFront cache invalidation**
    - Forceably clear cache before TTL expiration
    - Patterns: /*, /images/*, /blog/index.html
    - $0.005 per path after free tier (first 100/month)
    - Eventual consistency (propagates to all edges)

### Containers (8 Questions)

11. **Docker image layers and caching**
    - Each Dockerfile instruction = layer
    - Layers cached, reused if unchanged
    - Order matters (frequent changes at end)
    - Multi-stage build reduces image size

12. **Deployment vs StatefulSet in Kubernetes**
    - Deployment: Stateless, pods interchangeable
    - StatefulSet: Stateful, stable identity (nginx-0, 1)
    - Deployment: Web servers, microservices
    - StatefulSet: Databases, cache clusters

13. **Kubernetes Service types**
    - ClusterIP: Internal DNS only (default)
    - NodePort: Port on each node (31000-32767)
    - LoadBalancer: AWS ALB/NLB provisioned
    - ExternalName: CNAME to external service

14. **ECS EC2 vs Fargate**
    - EC2: You manage instances (control, cheaper)
    - Fargate: AWS manages (simpler, serverless)
    - EC2: Predictable sustained workloads
    - Fargate: Variable/burst workloads

15. **Container health checks purpose**
    - Kubernetes: Liveness (restart), Readiness (traffic)
    - Docker: HEALTHCHECK (status)
    - ECS: Application Load Balancer health check
    - Orchestrator replaces failing containers automatically

16. **Dockerfile multi-stage build benefit**
    - Build stage: Includes dependencies, dev tools
    - Final stage: Only runtime needed
    - Reduces image size significantly
    - Improves security (no build tools in production)

17. **Istio service mesh - what does it do?**
    - Manages service-to-service communication
    - Envoy sidecars (one per pod)
    - mTLS, traffic routing, circuit breaking
    - Observability: Automatic metrics, traces

18. **EKS security - key points**
    - RBAC: Service accounts, roles, role bindings
    - Network policies: Pod-to-pod firewall
    - Secrets: Encrypted in etcd
    - IAM: IRSA (workload identity)

### CI/CD (6 Questions)

19. **Blue-green vs Canary vs Rolling deployment**
    - Blue-green: Two environments, instant switch
    - Canary: Gradual traffic shift (5%, 25%, 100%)
    - Rolling: Replace 1 at a time
    - Trade-off: Cost vs risk vs speed

20. **CI vs CD**
    - CI: Automated build/test on every commit
    - CD: Automated deployment to prod
    - CI: Multiple times per day
    - CD: Per release with approval

21. **CodeBuild buildspec.yml phases**
    - pre_build: Install dependencies, login to registries
    - build: Compile, run tests, build artifacts
    - post_build: Push images, generate manifests
    - Each phase can use environment variables, secrets

22. **CloudFormation change sets**
    - Preview changes before applying
    - Shows what will be created/modified/deleted
    - Safer than direct stack update
    - Can review before executing

23. **Terraform state management**
    - Local state: For development
    - Remote (S3): For production, enable locking
    - State contains sensitive data (encrypt)
    - Never commit tfstate to Git

24. **Infrastructure drift detection**
    - Drift: Resources manually changed outside IaC
    - Detection: CloudFormation, Terraform
    - Why important: Ensures IaC is source of truth
    - Resolution: Sync or revert manual changes

### Operations & Advanced (6 Questions)

25. **RPO and RTO definitions**
    - RPO: Max acceptable data loss (Recovery Point Objective)
    - RTO: Max acceptable downtime (Recovery Time Objective)
    - Example: RPO 1hr, RTO 4hrs → Lose up to 1hr data, down max 4hrs

26. **Active-passive vs Active-active DR**
    - Active-passive: Primary active, secondary standby (5-30min RTO)
    - Active-active: Both regions active (instant RTO)
    - Active-passive: Simpler, cheaper, less tested
    - Active-active: Complex, expensive, always tested

27. **Reserved Instances vs Savings Plans**
    - RI: Specific instance type, 30-60% savings
    - SP: Flexible instance type, 20-30% savings
    - RI: Good if predictable + fixed
    - SP: Good if variety of types needed

28. **Least privilege principle**
    - Grant only minimum permissions needed
    - Start with deny, add specific allows
    - Use resource ARNs (not "*")
    - Add conditions (IP, time, etc.)

29. **Secrets rotation strategy**
    - Create new secret version
    - Update application (with compatibility)
    - Delete old after grace period
    - Automate with Secrets Manager

30. **Observability: Logs vs Metrics vs Traces**
    - Logs: Event details (strings, errors)
    - Metrics: Numeric measurements (CPU %, latency)
    - Traces: Request path through system
    - All three needed for complete observability

---

## 🎨 Architecture Decision Checklist

When designing architecture, always consider:

**Functional Requirements:**
- [ ] What does system need to do?
- [ ] User flow and interactions?
- [ ] Data consistency requirements?
- [ ] Transaction requirements?

**Non-Functional Requirements:**
- [ ] Uptime requirement (99.9%, 99.99%)?
- [ ] Response time acceptable (< 100ms)?
- [ ] Data retention (days, months, years)?
- [ ] Scale (concurrent users, requests/sec)?

**High Availability:**
- [ ] Multi-AZ deployment?
- [ ] Health checks for auto-recovery?
- [ ] Load balancing strategy?
- [ ] Circuit breaking/retry logic?

**Disaster Recovery:**
- [ ] RPO/RTO targets?
- [ ] Backup strategy?
- [ ] Multi-region?
- [ ] Tested recovery procedures?

**Security:**
- [ ] Least privilege IAM?
- [ ] Encryption (at rest, in transit)?
- [ ] Secrets management?
- [ ] Audit logging (CloudTrail)?
- [ ] Network isolation (VPC)?

**Operations:**
- [ ] Monitoring & alerting?
- [ ] Logging strategy?
- [ ] Deployment strategy?
- [ ] Runbooks for incidents?
- [ ] Infrastructure as Code?

**Cost:**
- [ ] Estimated monthly cost?
- [ ] Cost optimization opportunities?
- [ ] Reserved vs on-demand mix?
- [ ] Auto-scaling in place?

**Scalability:**
- [ ] How to scale each component?
- [ ] Bottlenecks identified?
- [ ] Database scaling (read replicas, sharding)?
- [ ] Caching strategy?

---

## 🔧 Common Tools & Commands

### AWS CLI Quick Ref

```bash
# Check current identity
aws sts get-caller-identity

# Create/list resources
aws ec2 describe-instances
aws s3 ls
aws rds describe-db-instances
aws ecs list-clusters

# CloudWatch logs
aws logs tail /log/group/name --follow

# View recent errors
aws logs filter-log-events \
  --log-group-name /app/logs \
  --filter-pattern "[ERROR]"

# Deploy CloudFormation
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml

# CodePipeline
aws codepipeline start-pipeline-execution --name my-pipeline
```

### Docker Commands

```bash
# Build multi-stage image
docker build -t my-app:1.0 -f Dockerfile .

# Scan for vulnerabilities
docker scan my-app:1.0

# Run with health check
docker run --health-cmd="curl localhost:8080/health" \
  --health-interval=30s my-app:1.0

# View layers
docker history my-app:1.0
```

### Kubernetes

```bash
# Deploy application
kubectl apply -f deployment.yaml

# View pods and logs
kubectl get pods -n production
kubectl logs pod-name -n production

# Scale deployment
kubectl scale deployment web --replicas=5

# Port forward for testing
kubectl port-forward svc/api 8080:80

# Check resource usage
kubectl top nodes
kubectl top pods

# RBAC
kubectl create serviceaccount app-sa
kubectl create role app-role --verb=get,list --resource=pods
kubectl create rolebinding app-binding \
  --serviceaccount=default:app-sa --role=app-role
```

### Terraform

```bash
# Validate configuration
terraform validate

# Plan changes
terraform plan -out=tfplan

# Apply changes
terraform apply tfplan

# Destroy resources
terraform destroy

# Check state
terraform state list
terraform state show aws_instance.web
```

---

## 📋 Troubleshooting Quick Flowchart

```
Problem Detected
    ↓
Check Status Page (AWS, services)
    ↓ (No status updates)
Check Recent Changes (deployments, config)
    ↓ (No recent changes)
Check CloudWatch Dashboard
    ├→ Metrics spiking? (CPU, memory, connection)
    ├→ Errors increasing?
    └→ Latency high?
    ↓
Use CloudWatch Logs
    ├→ Search for [ERROR]
    ├→ Look for exceptions
    └→ Check timestamps
    ↓
Use X-Ray Traces
    ├→ Identify slow segment
    ├→ Check dependencies
    └→ Find bottleneck
    ↓
Isolate Component
    ├→ Is it application?
    ├→ Is it infrastructure?
    └→ Is it dependent service?
    ↓
Fix (if known) or Escalate
    ├→ Quick mitigation (scale up, switch over)
    ├→ Long-term fix
    └→ Prevention measures
```

---

## 🎯 Interview Day Tips

### Morning Of
- [ ] Eat good breakfast (don't interview hungry)
- [ ] Light review of key concepts (30 min)
- [ ] Test technology (camera, mic, internet)
- [ ] Have notes ready for reference

### During Interview
- [ ] Smile and make eye contact (even video!)
- [ ] Think out loud (explain your reasoning)
- [ ] Ask clarifying questions
- [ ] Draw diagrams on whiteboard/screen share
- [ ] Don't interrupt interviewer
- [ ] Acknowledge feedback and adjust

### Phrases to Use
- "Let me think through this..." (okay to pause)
- "Can you clarify what you mean by...?" (shows understanding)
- "I would need to know more about..." (shows experience)
- "Here's a tradeoff to consider..." (shows architectural thinking)
- "Based on my experience with..." (reference real projects)

### Phrases to Avoid
- "I don't know" (better: "I haven't worked with that specifically but...")
- "That's easy/simple" (arrogant)
- "I've only used..." (limiting)
- Definitive answers without caveats (everything has tradeoffs)

---

## 📱 Last-Minute Cramming Guide (If You Have 2 Hours)

1. **Review the Big Picture (20 min)**
   - Read README-INTERVIEW-PREP.md introduction
   - Review the 30 essential questions above
   - Note areas you're weakest on

2. **Deep Dive on Weak Areas (60 min)**
   - Phase materials related to weak areas
   - Focus on concepts, not details
   - Read interview questions and answers

3. **Practice Out Loud (30 min)**
   - Pick 1 architecture scenario
   - Explain it out loud (as if to interviewer)
   - Time yourself (aim for 15 min)
   - Get comfortable with your explanations

4. **Confidence Building (10 min)**
   - Remember: You know more than you think
   - Recall your real projects
   - Prepare 2-3 good examples
   - Believe in your preparation

---

## 🎓 After You Get the Job

1. **Keep Learning**
   - AWS releases new services constantly
   - Subscribe to AWS newsletter
   - Follow DevOps blogs and YouTube channels

2. **Share Knowledge**
   - Mentor junior engineers
   - Write technical blog posts
   - Give talks at meetups
   - Help team improve processes

3. **Stay Current**
   - Recertify every 3 years
   - Read architecture blogs
   - Experiment with new tools
   - Build projects in free tier

4. **Contribute to Community**
   - Open source contributions
   - Answer Stack Overflow questions
   - Network with other engineers
   - Engage with DevOps community

---

## 💪 Final Motivation

**You've prepared comprehensively. You know this material.**

The questions will be hard, but you have:
- Deep understanding of AWS services
- Real architecture experience  
- Troubleshooting skills
- Communication prepared

Remember: Interviewers want you to succeed!

They're looking for someone who:
- ✓ Understands their systems
- ✓ Can operate them reliably
- ✓ Thinks about tradeoffs
- ✓ Communicates clearly
- ✓ Has real experience

**You've got this. Go ace that interview! 🚀**

---

**Print this guide. Reference during prep. Glance before interview.**

**Good luck! 🎉**
