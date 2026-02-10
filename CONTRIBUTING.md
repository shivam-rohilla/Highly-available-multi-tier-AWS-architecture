# Contributing to Highly Available Multi-Tier Architecture

Thank you for your interest in contributing to this project! This document provides guidelines and instructions for contributing.

## Table of Contents
- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Development Setup](#development-setup)
- [Contribution Workflow](#contribution-workflow)
- [Documentation Guidelines](#documentation-guidelines)
- [Testing Requirements](#testing-requirements)
- [Commit Message Guidelines](#commit-message-guidelines)

---

## Code of Conduct

### Our Pledge
We are committed to providing a welcoming and inclusive environment for all contributors, regardless of background or experience level.

### Our Standards
- Be respectful and constructive in communication
- Accept constructive criticism gracefully
- Focus on what's best for the project and community
- Show empathy towards other contributors

### Enforcement
Instances of unacceptable behavior may be reported to the project maintainers.

---

## How Can I Contribute?

### Reporting Bugs

Before creating a bug report:
1. Check the [Troubleshooting Guide](TROUBLESHOOTING.md)
2. Search existing issues to avoid duplicates
3. Verify the issue is reproducible

When submitting a bug report, include:
- Clear, descriptive title
- Steps to reproduce the issue
- Expected vs actual behavior
- Screenshots or logs (if applicable)
- Environment details (AWS region, instance types, etc.)
- AWS CLI version and other relevant versions

**Bug Report Template:**
```markdown
## Bug Description
[Clear description of the bug]

## Steps to Reproduce
1. Step 1
2. Step 2
3. Step 3

## Expected Behavior
[What you expected to happen]

## Actual Behavior
[What actually happened]

## Environment
- AWS Region: ap-south-1
- Instance Type: t3.micro
- RDS Version: MySQL 8.4.7
- AMI: Amazon Linux 2023

## Screenshots/Logs
[If applicable]

## Additional Context
[Any other relevant information]
```

### Suggesting Enhancements

Enhancement suggestions are welcome! Please include:
- Clear description of the proposed enhancement
- Rationale (why it would be useful)
- Potential implementation approach
- Any trade-offs or considerations

**Enhancement Template:**
```markdown
## Enhancement Description
[Clear description of the enhancement]

## Motivation
[Why this enhancement is needed]

## Proposed Solution
[How you propose to implement it]

## Alternatives Considered
[Other approaches you considered]

## Additional Context
[Any other relevant information]
```

### Documentation Improvements

Documentation contributions are highly valued:
- Fix typos or unclear explanations
- Add missing information
- Improve examples
- Update outdated information
- Add diagrams or visuals

### Code Contributions

Areas for contribution:
- Infrastructure as Code (Terraform/CloudFormation)
- Automation scripts
- Monitoring and alerting improvements
- Security enhancements
- Cost optimization
- Performance improvements

---

## Development Setup

### Prerequisites
- AWS CLI installed and configured
- Git
- Text editor or IDE
- Basic understanding of AWS services
- (Optional) Terraform or CloudFormation knowledge

### Setting Up Your Development Environment

1. **Fork the repository**
```bash
# Click "Fork" on GitHub
```

2. **Clone your fork**
```bash
git clone https://github.com/YOUR-USERNAME/highly-available-architecture.git
cd highly-available-architecture
```

3. **Add upstream remote**
```bash
git remote add upstream https://github.com/ORIGINAL-OWNER/highly-available-architecture.git
```

4. **Create a feature branch**
```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/bug-description
```

### AWS Environment Setup

For testing your changes:
1. Use a separate AWS account or region
2. Tag all resources with `Environment: Development`
3. Clean up resources after testing to avoid charges
4. Never commit AWS credentials

---

## Contribution Workflow

### 1. Create an Issue
Before starting work, create or comment on an issue describing what you plan to do.

### 2. Branch Naming
Use descriptive branch names:
- `feature/add-cloudwatch-dashboard`
- `fix/alb-health-check-timeout`
- `docs/update-troubleshooting-guide`
- `security/implement-waf-rules`

### 3. Make Your Changes
- Follow existing code style and patterns
- Keep changes focused and atomic
- Test thoroughly in your development environment
- Update documentation as needed

### 4. Test Your Changes
- Deploy changes to a test environment
- Verify all functionality works as expected
- Test rollback procedures
- Document any new testing procedures

### 5. Update Documentation
- Update README.md if adding new features
- Add entries to TROUBLESHOOTING.md for new issues
- Update SPECIFICATIONS.md for infrastructure changes
- Update CHANGELOG.md with your changes

### 6. Commit Your Changes
```bash
git add .
git commit -m "feat: add CloudWatch dashboard for monitoring"
```

### 7. Push to Your Fork
```bash
git push origin feature/your-feature-name
```

### 8. Create Pull Request
- Go to GitHub and create a Pull Request
- Fill out the PR template completely
- Link related issues
- Wait for review

---

## Documentation Guidelines

### Style Guide
- Use clear, concise language
- Write in present tense
- Use active voice
- Include code examples where helpful
- Add screenshots for UI-related documentation

### Markdown Standards
- Use proper heading hierarchy (h1 → h2 → h3)
- Include table of contents for long documents
- Use code blocks with language specification
- Use tables for structured data
- Add links to related documentation

### Code Examples
- Include complete, working examples
- Add comments explaining complex parts
- Show both AWS Console and CLI methods when applicable
- Include expected output

**Example:**
```bash
# Check Auto Scaling Group status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ha-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize]' \
  --output table

# Expected output:
# ---------------------------
# |  DesiredCapacity | 2    |
# |  MinSize         | 2    |
# |  MaxSize         | 4    |
# ---------------------------
```

---

## Testing Requirements

### Infrastructure Testing
Before submitting:
1. Deploy changes in test environment
2. Verify all components are healthy
3. Test failure scenarios
4. Verify monitoring and logging
5. Test rollback procedure
6. Clean up test resources

### Testing Checklist
- [ ] All services deployed successfully
- [ ] Health checks passing
- [ ] Application accessible
- [ ] Database connectivity working
- [ ] Auto Scaling responds to load
- [ ] Logs are being generated
- [ ] Metrics visible in CloudWatch
- [ ] Security groups properly configured
- [ ] No security vulnerabilities introduced
- [ ] Documentation updated
- [ ] Rollback tested

### Performance Testing
For performance-related changes:
- Conduct load testing
- Measure response times
- Monitor resource utilization
- Document performance improvements

---

## Commit Message Guidelines

### Format
```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `perf`: Performance improvements
- `test`: Adding or updating tests
- `chore`: Maintenance tasks
- `security`: Security improvements

### Scope
Optional, indicates what part of the project is affected:
- `alb`: Application Load Balancer
- `asg`: Auto Scaling Group
- `rds`: Database
- `vpc`: Networking
- `monitoring`: CloudWatch/logging
- `docs`: Documentation

### Examples
```
feat(monitoring): add CloudWatch dashboard for RDS metrics

Added comprehensive dashboard displaying:
- CPU utilization
- Database connections
- Read/Write latency
- Storage utilization

Closes #123
```

```
fix(alb): increase health check timeout

Health checks were failing due to short timeout.
Increased from 5s to 10s to accommodate slower responses.

Fixes #456
```

```
docs(troubleshooting): add section on database connection issues

Added detailed troubleshooting steps for:
- Security group configuration
- VPC connectivity
- Credential verification

Related to #789
```

---

## Pull Request Process

### PR Template
When creating a PR, include:

```markdown
## Description
[Describe your changes]

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Documentation update
- [ ] Performance improvement
- [ ] Security enhancement

## Testing
- [ ] Deployed to test environment
- [ ] All health checks passing
- [ ] Tested failure scenarios
- [ ] Tested rollback
- [ ] Updated documentation

## Screenshots
[If applicable]

## Checklist
- [ ] Code follows project style
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
- [ ] All tests passing
- [ ] No security vulnerabilities
- [ ] Reviewed own code

## Related Issues
Closes #[issue number]
```

### Review Process
1. Maintainers will review your PR
2. Address any requested changes
3. Once approved, maintainers will merge
4. Delete your feature branch after merge

### Review Criteria
- Code quality and style
- Test coverage
- Documentation completeness
- Security considerations
- Performance impact
- Cost implications

---

## Additional Resources

### AWS Documentation
- [VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
- [RDS User Guide](https://docs.aws.amazon.com/rds/)
- [ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)

### Best Practices
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)

### Tools
- [AWS CLI Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [CloudFormation Documentation](https://docs.aws.amazon.com/cloudformation/)

---

## Getting Help

If you need help:
1. Check existing documentation
2. Search closed issues
3. Ask in a new issue
4. Contact maintainers

---

## Recognition

Contributors will be recognized in:
- README.md contributors section
- Release notes
- Project documentation

Thank you for contributing! 🎉
