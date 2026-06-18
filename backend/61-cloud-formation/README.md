# CloudFormation vs Terraform

You've got your infrastructure defined as code. But you're standing at a crossroads: AWS CloudFormation or HashiCorp Terraform? Both do the same thing on paper—provision cloud resources from declarative templates. But they approach it very differently, and the choice impacts your workflow, team, and sanity.

Let's break it down.

---

## The Core Difference

**CloudFormation is AWS-native.** It's deeply integrated, free, and understands AWS resource dependencies implicitly. You write YAML or JSON templates, and CloudFormation handles the orchestration.

**Terraform is cloud-agnostic.** It talks to AWS, Azure, GCP, and hundreds of other providers through plugins. You write HCL (HashiCorp Configuration Language), and it manages state files to track what's deployed.

| Aspect | CloudFormation | Terraform |
|--------|---------------|-----------|
| Language | YAML / JSON | HCL |
| State management | Automatic (AWS-managed) | Local or remote state files |
| AWS support | Same-day support (AWS launches → CloudFormation support) | Lag days to weeks for new services |
| Multi-cloud | AWS only | AWS, Azure, GCP, Kubernetes, etc. |
| Cost | Free | Free (open-source), HCP Terraform paid tiers |
| Rollback | Automatic on failure | State-driven, needs manual recovery |
| Modularity | Nested stacks | Modules (simpler to share) |

---

## What CloudFormation Does Well

### Deep AWS Integration

CloudFormation knows AWS resources inside out. When you define an EC2 instance and a security group, CloudFormation understands the dependency without you spelling it out.

```yaml
# CloudFormation: dependencies are implicit
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      SecurityGroups:
        - !Ref MySecurityGroup  # CloudFormation resolves this automatically
  
  MySecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH
```

### ✅ The good stuff

- **No state file to manage** — AWS tracks the stack state. You don't lose track of what's deployed.
- **Automatic rollback on failure** — stack creation fails? CloudFormation rolls back and cleans up. Terraform leaves partial state you have to clean manually.
- **IAM integration** — you can grant/restrict CloudFormation actions just like any other AWS service.
- **StackSets** — deploy the same template across multiple accounts and regions in one operation.
- **Drift detection** — built-in. Terraform requires `terraform plan` to detect differences.

### ❌ The pain points

```yaml
# CloudFormation: verbose for complex logic
Conditions:
  IsProduction: !Equals [!Ref Environment, "production"]
  
Resources:
  ProductionInstance:
    Condition: IsProduction
    Type: AWS::EC2::Instance
```

Conditionals and loops in YAML get ugly fast. You end up using `!Sub`, `!Join`, and `!If` functions that look like someone's trying to write Lisp in YAML.

- YAML/JSON only — no proper programming constructs
- Stack limits (200 resources per stack, workaround: nested stacks)
- Slow feedback loop — template validation is okay, but actual errors surface during deployment

---

## What Terraform Does Well

### Multi-cloud and Provider Ecosystem

Need to provision AWS resources, configure a Cloudflare DNS record, and deploy a Kubernetes cluster in one workflow? Terraform's provider model makes this trivial.

```hcl
# Terraform: one workflow across providers
provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"
}

resource "cloudflare_record" "www" {
  zone_id = var.cloudflare_zone_id
  name    = "www"
  value   = aws_instance.web.public_ip
  type    = "A"
}
```

One `terraform apply` and everything happens in the right order.

### ✅ The good stuff

- **HCL is far more readable** — proper blocks, expressions, and `for_each` loops
- **Plan output** — `terraform plan` shows exactly what will change before you apply
- **Modules** — share and version infrastructure like packages
- **State locking** — prevents concurrent modifications (with remote backends)
- **Rich ecosystem** — hundreds of providers beyond cloud (DNS, monitoring, databases)

### ❌ The pain points

```
# Terraform: state file is a single point of failure (if not managed properly)
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

- **State management is your problem** — lose your state file or corrupt it, and you're in a bad place
- **Provider lag** — when AWS launches a new service, Terraform needs a provider update. You might wait weeks.
- **Partial apply failures** — Terraform fails mid-apply and leaves partial state. You need to understand the state to fix it.
- **No automatic rollback** — you have to manually `terraform apply` the previous known-good config

---

## When to Pick Each

### Choose CloudFormation when...

- **You're all-in on AWS** — no multi-cloud plans
- **You want minimal operational overhead** — no state management, no extra CI/CD tooling
- **You need same-day support for new AWS services**
- **You're in a heavily regulated environment** — CloudFormation's IAM integration makes audit trails easier
- **You have AWS Organizations + multiple accounts** — StackSets are genuinely good

### Choose Terraform when...

- **You're multi-cloud or hybrid** — AWS + GCP + on-prem = Terraform territory
- **You want to manage infrastructure + configurations in one tool** — AWS resources plus Cloudflare DNS, Datadog monitors, GitHub repos
- **You prioritize readability and modularity** — HCL with modules is cleaner than nested CloudFormation stacks
- **Your team already uses other HashiCorp tools** — Vault, Consul, Nomad
- **You want a larger community** — Terraform's modules registry has thousands of ready-to-use modules

---

## The Hybrid Approach

Here's the dirty secret: **you don't have to pick one.** Many teams use both.

```
Scenario:
├── Core networking (VPC, subnets, NAT) → CloudFormation
├── Application infrastructure (ECS, RDS) → Terraform
└── CI/CD pipeline → CloudFormation (CodePipeline, CodeBuild)
```

CloudFormation handles the static, foundational stuff that rarely changes (VPCs, IAM roles). Terraform handles the application-level resources that need modularity and multi-provider orchestration.

It adds complexity, sure. But it lets each tool do what it's best at.

---

## Bottom Line

**CloudFormation is the default choice for AWS-only teams that want managed state and deep integration.** No extra tooling, no state file anxiety, no provider lag.

**Terraform is the choice for teams that value flexibility, readability, and multi-cloud capability.** It's more work to manage state, but the payoff in modularity and provider ecosystem is massive.

Don't overthink this one. If you're AWS-only and don't plan to switch, CloudFormation is perfectly fine. If you touch anything outside AWS (or think you will), Terraform wins by a mile.

---

## Key Takeaways

- CloudFormation handles state automatically; Terraform makes you manage it (use S3 + DynamoDB for remote state + locking)
- CloudFormation has tighter AWS integration and instant new-service support
- Terraform wins for multi-cloud, readability, and modularity with HCL
- Partial failures hurt more in Terraform because of residual state
- Both tools can coexist — use each where it shines
- For state management with Terraform, always use remote backends with locking — never local state for production
