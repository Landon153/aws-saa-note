# Cantrill Review – Infrastructure as Code

## CloudFormation Physical and Logical Resources
- **CloudFormation Template**: Written in YAML or JSON.
  - Defines **logical resources** – the “WHAT” (e.g., desired infrastructure components).
- **Templates** are used to create **Stacks**.
- **Stacks**:
  - Can create 1 stack, multiple stacks (e.g., 100 stacks), or multiple stacks per region (e.g., 20 stacks per region).
  - Stacks create **physical resources** based on the logical resources defined in the template.
- **Deletion**: When a stack is deleted, the associated physical resources are typically deleted.