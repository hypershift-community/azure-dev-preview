# HyperShift Azure Developer Preview

This repository provides resources for the HyperShift self-managed Azure developer preview release.

## What is HyperShift?

HyperShift enables you to deploy and manage OpenShift hosted control planes on a management cluster. For self-managed Azure deployments, this means:

- **Reduced costs**: Run multiple OpenShift control planes as pods on a shared management cluster
- **Improved density**: Host dozens of control planes on the same infrastructure
- **Simplified management**: Centralize control plane operations while isolating workload data planes
- **Increased flexibility**: Choose your own management cluster platform and customize networking

## Documentation

All documentation is available in the `docs/` directory.

To build and serve documentation locally:

```bash
cd docs
make serve-containerized
```

Then visit http://localhost:8000

## Support and Feedback

This is a developer preview release. For questions, issues, or feedback:

1. Review the troubleshooting guide in the documentation
2. Check [HyperShift GitHub Issues](https://github.com/openshift/hypershift/issues) for known issues
3. For preview-specific questions, open an issue in this repository

## Contributing

This repository is focused on Azure developer preview resources. For contributions to the main HyperShift project, see:
- [HyperShift GitHub Repository](https://github.com/openshift/hypershift)
- [HyperShift Documentation](https://hypershift.pages.dev)

## License

This project is licensed under the Apache License 2.0 - see the upstream [HyperShift project](https://github.com/openshift/hypershift) for details.
