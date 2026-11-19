# Deploying Terraform Workloads Using Stack-Based Pipelines

## Overview

Stack-based pipelines provide a scalable approach for managing multiple independent infrastructure concerns within the same environment/provider/region. This pattern is particularly useful for large centralized deployment projects that need to partition resources by domain, workload, or organizational boundaries.

## When to Use Stack-Based Pipelines

**Use stacks when you have:**

- Independent infrastructure concerns with no dependencies between them
- Multiple domains or workloads managed by different teams
- Need for isolated Terraform state files to reduce contention
- Desire to reduce blast radius of changes
- Requirements for domain-specific code review and ownership

**Examples of good stack boundaries:**

- `core` - foundational shared resources (legacy container)
- `monitoring` - observability and alerting infrastructure
- `compliance` - security scanning and compliance tools
- `identity` - user and access management systems
- `data-engineering` - data processing and analytics workloads
- `kubernetes` - container orchestration resources
- `security` - security tooling and integrations

## Implementation Example

### Directory Structure

```txt
environments/
  test/
    multi/
      global/
        core/                    # Legacy stack
          main.tf
          backend.tf
          terraform.tfvars
        monitoring/              # New stack
          main.tf
          backend.tf
          terraform.tfvars
        kubernetes/              # New stack
          main.tf
          backend.tf
          terraform.tfvars
  prod/
    multi/
      global/
        core/
        monitoring/
        kubernetes/
modules/
  multi-global-core/            # Stack-specific modules
  multi-global-monitoring/
  multi-global-kubernetes/
```

## Parent-Child Pipeline Architecture

Stack-based deployments benefit from GitLab's parent-child pipeline architecture to improve scalability and performance. This approach uses a lightweight parent pipeline to dynamically trigger stack-specific child pipelines based on changes.

### Architecture Overview

1. **Parent Pipeline**: A lightweight coordinator that:

   - Determines which stacks have changes
   - Triggers child pipelines only for affected stacks
   - Handles global validation and checks

2. **Child Pipelines**: Stack-specific pipelines that:
   - Run terraform operations for a single stack
   - Execute in parallel with other stack pipelines
   - Contain the complete deployment workflow for one stack

### Benefits of Parent-Child Architecture

- **Improved Performance**: Only affected stacks are processed
- **Reduced Pipeline Duration**: Parallel execution of independent stacks
- **Better Resource Utilization**: Runner resources allocated only to changed stacks
- **Clearer Pipeline Visualization**: Each stack has its own pipeline graph
- **Simplified Debugging**: Failures isolated to specific stack pipelines

## Real-World Example: Asset Inventory Resource Deployment

For a production implementation of stack-based pipelines using parent-child architecture, refer to the Asset Inventory resource deployment project. This project demonstrates:

- Multiple independent stacks
- Dynamic pipeline generation using templates
- Stack-specific trigger rules based on file changes
- Complete parent-child pipeline implementation

## Best Practices

### Stack Independence

- **No cross-stack dependencies**: Each stack should be completely independent
- **Separate state files**: Use unique backend keys for each stack
- **Independent modules**: Create dedicated modules per stack when possible

### Naming Conventions

- **Stack names**: Use long-lived organizational boundaries (departments, workloads)
- **Avoid team names**: Teams change, but organizational boundaries are more stable
- **Avoid infrastructure types**: Don't use "database", "networking" - these decompose workloads by infrastructure concern
- **Backend keys**: Follow pattern `{provider}/{region}/{stack}`

### Code Organization

- **CODEOWNERS**: Assign domain experts to review their specific stacks
- **Module structure**: Keep stack-specific logic in dedicated modules
- **Variable isolation**: Use stack-specific terraform.tfvars files

### Pipeline Optimization

- **Parallel execution**: Stacks run independently and can execute in parallel
- **Resource efficiency**: Smaller state files improve Terraform performance
- **Selective execution**: Use stack-trigger-rules to run pipelines only when relevant files change

## Adding a New Stack

Follow these steps to add a new stack to an existing stack-based deployment:

1. **Create the directory structure**:

   ```bash
   # Create environment directories
   mkdir -p environments/test/<provider>/<region>/<new-stack>
   mkdir -p environments/prod/<provider>/<region>/<new-stack>

   # Create module directory
   mkdir -p modules/<provider>-<region>-<new-stack>
   ```

2. **Create the stack module**:

   - Implement the stack-specific resources in `modules/<provider>-<region>-<new-stack>/`
   - Create standard module files (main.tf, variables.tf, outputs.tf, etc.)

3. **Create the environment configurations**:

   - Create backend.tf with a unique state key for the stack
   - Create main.tf that references the stack module
   - Add stack-specific variables in terraform.tfvars

4. **Update the parent pipeline**:

   - Add stack-trigger-rules component for the new stack

   ```yml
   - component: $CI_SERVER_FQDN/engineering/dev-tools/internal/gitlab-cicd-deployment-workflow/stack-trigger-rules@2
     inputs:
       stack: <new-stack>
       changes-paths:
         - "environments/*/<provider>/<region>/<new-stack>/**/*.tf"
         - "modules/<provider>-<region>-<new-stack>/**/*.tf"
   ```

5. **Add pipeline generation and trigger jobs**:
   ```yml
   trigger-<new-stack>:
     stage: stack-triggers
     extends: .stack-trigger-rules:<new-stack>
     trigger:
       include:
         - local: .gitlab/stack-pipeline.gitlab-ci.yml
           inputs:
             stack: <new-stack>
   ```

## Migration Strategy

### From Monolithic to Stack-Based

1. **Identify boundaries**: Analyze existing resources for natural separation points by workload/domain
2. **Create new stacks**: Start with new functionality in dedicated stacks
3. **Gradual migration**: Move existing resources to appropriate stacks over time
4. **Legacy container**: Keep complex migrations in a temporary "core" or "legacy" stack

### Example Migration Path

```txt
Before (Monolithic):
environments/test/multi/global/
  ├── main.tf (100+ resources across multiple workloads)
  └── terraform.tfvars

After (Stack-Based):
environments/test/multi/global/
  ├── core/              # Legacy resources during migration
  ├── monitoring/        # Observability workload
  ├── identity/          # Identity management workload
  ├── data-engineering/  # Data processing workload
  └── kubernetes/        # Container orchestration workload
```

## Troubleshooting

### Common Issues

**State Lock Conflicts:**

- Ensure each stack has a unique backend key
- Verify no shared state files between stacks

**Cross-Stack Dependencies:**

- Review architecture - stacks should be independent
- Consider if resources belong in the same stack
- Use data sources for read-only references if absolutely necessary

**Pipeline Performance:**

- Monitor job execution times per stack
- Consider further stack subdivision if individual stacks become too large
- Leverage parallel execution capabilities

## Related Documentation

- [Standard Terraform Deployment Structure](https://github.com/colchuck-ai/workload-handbook/blob/main/tools/terraform/terraform.md)
- [Terraform Stack Decision Record](https://github.com/colchuck-ai/workload-handbook/blob/main/decision-records/use-terraform-stacks-for-independent-concern-partitioning.md)
- [GitLab CI/CD Components Documentation](https://docs.gitlab.com/ci/components/)
