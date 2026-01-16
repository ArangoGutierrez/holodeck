# Feature Specifications (Prompts)

This directory contains detailed feature specifications for planned Holodeck
enhancements. These documents serve as design specifications and implementation
guides for new functionality.

## Documents

| File | Description | Status |
|------|-------------|--------|
| [kubernetes-from-source.md](kubernetes-from-source.md) | Install Kubernetes from git refs (commit, branch, tag, fork) | Proposed |

## Purpose

These specification documents ("prompts") are intended to:

1. **Define requirements** before implementation begins
2. **Research existing solutions** (KIND, Minikube, etc.) for best practices
3. **Propose API schemas** with concrete type definitions
4. **Provide examples** showing expected YAML configurations
5. **Plan implementation** with phased approaches
6. **Guide testing** with validation rules and error handling

## Related Issues

- [Epic: Provision Core Dependencies from Multiple Sources](https://github.com/NVIDIA/holodeck/issues/567)
- [Epic: NVIDIA Container Toolkit Installation from Multiple Sources](https://github.com/NVIDIA/holodeck/issues/566)

## Contributing

When adding a new specification:

1. Create a new markdown file with a descriptive name
2. Follow the structure of existing specifications
3. Include research from existing tools where applicable
4. Provide concrete code examples (Go types, YAML configs)
5. Update this README with a link to the new document
