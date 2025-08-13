# Custom UV Buildpack for Cloud Foundry

⚠️ **EXPERIMENTAL** - This buildpack is experimental and not production-ready. Use at your own risk.

## Overview

This is a custom Cloud Foundry buildpack that uses [uv](https://github.com/astral-sh/uv) as the Python package manager instead of pip. UV is a fast Python package installer and resolver written in Rust.

## Features

- **Fast dependency resolution** using uv
- **Virtual environment isolation** for each application
- **Dynamic dependency parsing** from `pyproject.toml`
- **JupyterLab support** out of the box
- **Clean separation** between staging and runtime phases

## Architecture

### Buildpack Structure
```
bin/
├── detect      # Detects Python applications with pyproject.toml
├── compile     # Staging phase: copies files and creates startup script
└── release     # Provides startup command metadata
```

### Runtime Flow
1. **Install uv** in runtime environment
2. **Create virtual environment** using `uv venv`
3. **Parse dependencies** from `pyproject.toml`
4. **Install dependencies** using `uv pip install`
5. **Start JupyterLab** on the specified port

## Usage

### Prerequisites
- Cloud Foundry environment
- Python application with `pyproject.toml`

### Deployment
```bash
# Deploy using the custom buildpack
cf push cf-jupyterlab -b https://github.com/yannicklevederpvtl/uv-buildpack.git
```

### Application Structure
Your application should have:
```
your-app/
├── pyproject.toml    # Python project configuration
├── uv.lock          # Locked dependency versions (optional)
├── jupyter_notebook_config.py  # Jupyter configuration
└── other files...
```

### Example pyproject.toml
```toml
[project]
name = "cf-jupyterlab"
version = "0.1.0"
description = "JupyterLab on Cloud Foundry with uv"
requires-python = ">=3.11"
dependencies = [
    "jupyterlab==4.2.5",
    "notebook",
    "ipykernel",
    "python-dotenv",
    # ... other dependencies
]

[tool.uv]
dev-dependencies = []
```

## Configuration

### Environment Variables
- `JUPYTER_ENABLE_LAB`: Set to "yes" to enable JupyterLab
- `UV_CACHE_DIR`: Directory for uv cache (default: "/tmp/uv-cache")
- `UV_INDEX_URL`: PyPI index URL (default: "https://pypi.org/simple/")

### Health Checks
The buildpack supports port-based health checks for faster startup:
```yaml
health-check-type: port
```

## Experimental Status

⚠️ **This buildpack is experimental and has the following limitations:**

### Known Issues
- **Long startup time**: Dependencies are installed at runtime, not during staging
- **No dependency caching**: Each instance installs dependencies independently
- **Limited testing**: Not thoroughly tested in production environments
- **Version compatibility**: May not work with all Python package combinations

### Experimental Features
- **Runtime dependency installation**: Dependencies installed during application startup
- **Dynamic pyproject.toml parsing**: Dependencies extracted at runtime
- **Custom startup script generation**: Startup script created during staging

### Production Considerations
- **Startup latency**: First startup takes longer due to dependency installation
- **Resource usage**: Each instance requires more memory/disk for dependency installation
- **Network dependency**: Requires internet access during startup for package downloads
- **No buildpack caching**: Staging phase doesn't cache dependencies

## Development

### Building the Buildpack
```bash
# Clone the buildpack repository
git clone https://github.com/yannicklevederpvtl/uv-buildpack.git
cd uv-buildpack

# Make scripts executable
chmod +x bin/detect bin/compile bin/release

# Test locally (if possible)
```

### Customization
The buildpack can be customized by modifying:
- `bin/compile`: Staging phase logic
- `bin/detect`: Application detection logic
- `bin/release`: Startup command configuration

## Troubleshooting

### Common Issues

1. **Startup script not found**
   - Ensure the buildpack creates the startup script in the correct location
   - Check that the release script points to the right path

2. **Dependencies not installing**
   - Verify `pyproject.toml` format is correct
   - Check that dependencies are listed in the `dependencies` array
   - Ensure internet connectivity during startup

3. **Long startup times**
   - This is expected behavior for the experimental runtime installation approach
   - Consider using a different buildpack for production workloads

4. **Permission errors**
   - Ensure buildpack scripts have execute permissions
   - Check that the application has write permissions in `/home/vcap/app`

### Debugging
Enable verbose logging by checking Cloud Foundry application logs:
```bash
cf logs cf-jupyterlab --recent
```

## Alternatives

For production use, consider:
- **Official Python buildpack**: More stable and well-tested
- **Paketo Python buildpack**: Modern buildpack with better caching
- **Custom buildpack with staging installation**: Install dependencies during staging phase

## Contributing

⚠️ **This is an experimental project.** Contributions are welcome but should be clearly marked as experimental.

### Development Guidelines
- Test thoroughly before submitting changes
- Document experimental features clearly
- Consider backward compatibility
- Add appropriate warnings for experimental functionality

## License

This buildpack is experimental and provided as-is. Use at your own risk.

## Acknowledgments

- [UV](https://github.com/astral-sh/uv) - Fast Python package manager
- [Cloud Foundry](https://www.cloudfoundry.org/) - Platform for running applications
- [JupyterLab](https://jupyterlab.readthedocs.io/) - Web-based interactive development environment

---

⚠️ **EXPERIMENTAL** - This buildpack is not recommended for production use. Use at your own risk.
