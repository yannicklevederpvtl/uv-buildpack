# Custom UV Buildpack for Cloud Foundry

⚠️ **EXPERIMENTAL** - This buildpack is experimental and not production-ready. Use at your own risk.

## Overview

This is a custom Cloud Foundry buildpack that uses [uv](https://github.com/astral-sh/uv) as the Python package manager instead of pip. UV is a fast Python package installer and resolver written in Rust.

## Features

- **Fast dependency resolution** using uv
- **Virtual environment isolation** for each application
- **Multiple dependency formats** support (`pyproject.toml`, `requirements.txt`)
- **Python version specification** via `runtime.txt`
- **Flexible start commands** via manifest or environment variables
- **JupyterLab support** out of the box with automatic fallback
- **Universal Python app support** (Flask, Django, FastAPI, etc.)
- **Clean separation** between staging and runtime phases

## Architecture

### Buildpack Structure
```
bin/
├── detect      # Detects Python applications with pyproject.toml, requirements.txt, or uv.lock
├── compile     # Staging phase: installs uv and prepares startup script
└── release     # Provides startup command metadata
```

### Runtime Flow
1. **Install uv** in runtime environment
2. **Create virtual environment** with specified Python version
3. **Install dependencies** using uv
4. **Activate virtual environment**
5. **Start application** with installed dependencies

## Usage

### Prerequisites
- Cloud Foundry environment
- Python application with `pyproject.toml` or `requirements.txt`

### Example Applications

#### Flask Application
```python
# app.py
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from Flask on Cloud Foundry with UV!'

if __name__ == '__main__':
    port = int(os.getenv('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

```toml
# pyproject.toml
[project]
name = "flask-app"
version = "0.1.0"
dependencies = [
    "flask",
    "gunicorn"
]
```

```yaml
# manifest.yml
applications:
- name: flask-app
  env:
    START_COMMAND_OVERRIDE: "gunicorn app:app --bind 0.0.0.0:$PORT"
```

#### Django Application
```toml
# pyproject.toml
[project]
name = "django-app"
version = "0.1.0"
dependencies = [
    "django",
    "gunicorn",
    "whitenoise"
]
```

```yaml
# manifest.yml
applications:
- name: django-app
  env:
    START_COMMAND_OVERRIDE: "gunicorn myproject.wsgi:application --bind 0.0.0.0:$PORT"
```

#### FastAPI Application
```toml
# pyproject.toml
[project]
name = "fastapi-app"
version = "0.1.0"
dependencies = [
    "fastapi",
    "uvicorn"
]
```

```bash
# Deploy with manifest command
cf push fastapi-app -f manifest.yml
```

### Deployment
```bash
# Deploy using the custom buildpack
cf push cf-jupyterlab -b https://github.com/yannicklevederpvtl/uv-buildpack.git

# Deploy with manifest file
cf push myapp -b https://github.com/yannicklevederpvtl/uv-buildpack.git -f manifest.yml

# Deploy with manifest file
cf push -f manifest.yml

# Generate lock file for reproducible builds (recommended)
uv lock
cf push -f manifest.yml
```

### Application Structure
Your application should have:
```
your-app/
├── pyproject.toml    # Python project configuration (primary)
├── requirements.txt  # Python dependencies (alternative)
├── uv.lock          # Locked dependency versions (recommended for production)
├── runtime.txt      # Python version specification (optional)

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

### Example requirements.txt
```
jupyterlab==4.2.5
notebook
ipykernel
python-dotenv
# ... other dependencies
```

### Example runtime.txt
```
python-3.11.x
```



## Configuration

### Environment Variables
- `JUPYTER_ENABLE_LAB`: Set to "yes" to enable JupyterLab

- `PYTHON_VERSION_OVERRIDE`: Override Python version (if not specified in runtime.txt)
- `START_COMMAND_OVERRIDE`: Override the default start command
- `CF_START_COMMAND`: Start command from CF CLI (internal use)

### Start Command Configuration

⚠️ **Important**: This buildpack does not support Procfiles, CF CLI commands (`-c` flag), or manifest `command` fields due to timing conflicts with Cloud Foundry's startup process.

#### Unsupported Methods
- **Procfiles**: Not supported (Cloud Foundry processes before dependencies are installed)
- **CF CLI commands** (`cf push -c "command"`): Not supported (same timing issue)
- **Manifest `command` field**: Not supported (same timing issue)

#### Supported Methods
The buildpack supports the following ways to specify how your application should start. Commands are evaluated in the following priority order:

#### 1. START_COMMAND_OVERRIDE Environment Variable
Set the start command via environment variable in your `manifest.yml`:
```yaml
applications:
- name: myapp
  env:
    START_COMMAND_OVERRIDE: "gunicorn myapp:app --bind 0.0.0.0:$PORT"
```



#### 2. JupyterLab Fallback (Default)
If no start command is specified and `jupyterlab` is in your dependencies, the buildpack will automatically start JupyterLab:
```bash
jupyter lab --ip=0.0.0.0 --port=$PORT --no-browser --allow-root --NotebookApp.token='' --NotebookApp.password=''
```

#### 3. Error (No Command Found)
If no start command is specified and `jupyterlab` is not in dependencies, the buildpack will show an error with helpful instructions.

#### Why These Limitations Exist
Cloud Foundry processes Procfiles, CF CLI commands, and manifest `command` fields **before** our buildpack's startup script runs. This means these commands would execute without the virtual environment and dependencies being available, causing "command not found" errors.

Our buildpack uses environment variables because they are processed by our startup script **after** dependencies are installed.

### UV Optimization Features

#### Lock File Support
For reproducible builds and faster dependency installation, include a `uv.lock` file:

```bash
# Generate lock file locally
uv lock

# The buildpack will automatically use the lock file for faster, reproducible installs
```

#### Performance Optimizations
The buildpack includes several optimizations for Cloud Foundry:

- **Lock File Usage**: ~50-70% faster dependency installation when `uv.lock` is present
- **Runtime Dependency Installation**: Dependencies installed during application startup
- **Python Version Management**: Automatic Python version installation if not available
- **Reproducible Builds**: Exact dependency versions with lock files



### Health Checks
The buildpack supports port-based health checks for faster startup:
```yaml
health-check-type: port
```

## Experimental Status

⚠️ **This buildpack is experimental and has the following limitations:**

### Known Issues
- **Limited testing**: Not thoroughly tested in production environments
- **Version compatibility**: May not work with all Python package combinations
- **Startup time**: Dependencies are installed during runtime, which may increase startup time

### Experimental Features
- **Runtime dependency installation**: Dependencies installed during application startup phase
- **Lock file support**: Reproducible builds with `uv.lock` files
- **Enhanced Python version management**: Automatic Python version installation
- **Flexible start commands**: Support for environment variables

### Production Considerations
- **Reliable startup**: Dependencies installed fresh in runtime environment
- **Consistent environments**: Each instance gets its own virtual environment
- **Longer startup times**: Dependency installation happens during application startup
- **Internet dependency**: Requires internet access during startup for dependency installation

## Best Practices for Production

### Performance Optimization
1. **Use Lock Files**: Generate `uv.lock` files for reproducible builds
   ```bash
   uv lock
   git add uv.lock
   git commit -m "Add lock file for reproducible builds"
   ```

2. **Specify Python Version**: Use `runtime.txt` for consistent Python versions
   ```
   # runtime.txt
   python-3.11.x
   ```

3. **Use environment variables**: Define explicit start commands for better control
   ```yaml
   # manifest.yml
   applications:
   - name: myapp
     env:
       START_COMMAND_OVERRIDE: "gunicorn myapp:app --bind 0.0.0.0:$PORT --workers 4"
   ```

### Monitoring and Debugging
- **Check Logs**: Monitor application logs for startup performance
- **Startup Times**: Track startup performance with and without lock files
- **Dependency Installation**: Monitor dependency installation times

## Important Notes

⚠️ **Start Command Limitations**: This buildpack has specific limitations on how start commands can be specified:

- **No Procfile Support**: Procfiles are not supported due to timing conflicts with Cloud Foundry's startup process
- **No CF CLI Commands**: CF CLI commands (`cf push -c "command"`) are not supported due to timing conflicts  
- **No Manifest Commands**: The manifest `command` field is not supported due to timing conflicts
- **Environment Variables Only**: Use `START_COMMAND_OVERRIDE` in manifest.yml for start commands
- **Auto-detection**: JupyterLab apps will auto-start if no explicit command is provided

These limitations exist because Cloud Foundry processes Procfiles, CF CLI commands, and manifest `command` fields **before** our buildpack's startup script runs, which would cause "command not found" errors since dependencies aren't installed yet.

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
   - Verify the dependency files exist in the application directory
   - Check that the virtual environment was created successfully
   - Ensure the runtime environment has internet access

3. **Long startup times**
   - Use lock files to speed up dependency installation
   - Consider pre-installing common dependencies

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
- **Paketo Python buildpack**: Modern buildpack with comprehensive features
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
- [JupyterLab](https://jupyterlab.readthedocs.io/) - Web-based interactive development environment

---

⚠️ **EXPERIMENTAL** - This buildpack is not recommended for production use. Use at your own risk.
# Fixed default JupyterLab command to use uv run
