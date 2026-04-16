# Dockerfile Migration Guide for setuptools 82.0.1

## Overview

This guide addresses the build failure caused by upgrading to setuptools 82.0.1, which deprecated `setup.py develop`. The CIS Docker build needs to be updated to use modern PEP 517 installation methods.

---

## Error Summary

```
error: subprocess-exited-with-error
× python setup.py develop did not run successfully.

DevelopDeprecationWarning: develop command is deprecated.
Please avoid running ``setup.py`` and ``develop``.
Instead, use standards-based tools like pip or uv.
```

**Root Cause**: setuptools 82.x removed support for `setup.py develop`. Editable installs must now use `pip install -e .` with PEP 517.

---

## Required Dockerfile Changes

### Change 1: Replace Editable Installs

**OLD (Breaking):**
```dockerfile
# In requirements.txt or Dockerfile
RUN cd /app/src/f5-cccl && python setup.py develop
RUN cd /app/src/f5-icontrol-rest && python setup.py develop
RUN cd /app/src/f5-ctlr-agent && python setup.py develop
```

**NEW (setuptools 82+ compatible):**
```dockerfile
# Use pip with --break-system-packages for system Python
RUN pip3 install --no-cache-dir --break-system-packages -e /app/src/f5-cccl
RUN pip3 install --no-cache-dir --break-system-packages -e /app/src/f5-icontrol-rest
RUN pip3 install --no-cache-dir --break-system-packages -e /app/src/f5-ctlr-agent
```

### Change 2: Update requirements.txt Format

If using `-e` in requirements.txt:

**OLD:**
```
-e git+https://github.com/f5devcentral/f5-cccl.git@branch#egg=f5-cccl
```

**NEW:**
```
f5-cccl @ git+https://github.com/f5devcentral/f5-cccl.git@branch
```

Or use local editable install:
```
-e /app/src/f5-cccl
-e /app/src/f5-icontrol-rest
-e /app/src/f5-ctlr-agent
```

### Change 3: Handle PEP 668 (Externally Managed Environments)

Ubuntu 24.04 enforces PEP 668, preventing system-wide pip installs without `--break-system-packages`.

**Solution Options:**

**Option A: Use --break-system-packages (Current approach)**
```dockerfile
RUN pip3 install --no-cache-dir --break-system-packages -e /app/src/f5-cccl
```

**Option B: Create a virtual environment (Recommended)**
```dockerfile
# Create venv
RUN python3 -m venv /app/.venv

# Use venv for all pip commands
RUN /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-cccl
RUN /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-icontrol-rest
RUN /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-ctlr-agent

# Update PATH to use venv
ENV PATH="/app/.venv/bin:$PATH"
```

---

## Complete Dockerfile Example

### Before (setuptools 79.0):
```dockerfile
RUN /opt/f5/apt-scripts/f5-apt-install \
        python3 \
        python3-pip \
        python3-dev \
        git \
        gcc \
        libffi-dev \
    && pip3 install --no-cache-dir --break-system-packages --ignore-installed -r /tmp/requirements.txt \
    && rm -rf /tmp/requirements.txt \
    && ln -sf /usr/bin/python3 /usr/local/bin/python \
    && apt-get remove -y git python3-pip python3-wheel python3-dev gcc libffi-dev \
    && apt-get autoremove -y \
    && /opt/f5/apt-scripts/f5-apt-clean \
    && chown -R ctlr:ctlr "$APPPATH" \
    && rm -f /bin/sh /bin/bash /bin/dash /bin/perl /usr/bin/perl
```

### After (setuptools 82.0.1):
```dockerfile
# Install build dependencies
RUN /opt/f5/apt-scripts/f5-apt-install \
        python3 \
        python3-pip \
        python3-dev \
        python3-venv \
        git \
        gcc \
        libffi-dev

# Create virtual environment
RUN python3 -m venv /app/.venv

# Install Python dependencies
COPY setup_requirements.txt /tmp/
RUN /app/.venv/bin/pip install --no-cache-dir --upgrade pip setuptools wheel \
    && /app/.venv/bin/pip install --no-cache-dir -r /tmp/setup_requirements.txt

# Install f5 packages as editable (for development) using modern method
COPY src/f5-cccl /app/src/f5-cccl
COPY src/f5-icontrol-rest /app/src/f5-icontrol-rest
COPY src/f5-ctlr-agent /app/src/f5-ctlr-agent

RUN /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-cccl \
    && /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-icontrol-rest \
    && /app/.venv/bin/pip install --no-cache-dir -e /app/src/f5-ctlr-agent

# Update PATH to use venv
ENV PATH="/app/.venv/bin:$PATH"

# Create python symlink
RUN ln -sf /app/.venv/bin/python3 /usr/local/bin/python

# Cleanup
RUN apt-get remove -y git python3-dev gcc libffi-dev \
    && apt-get autoremove -y \
    && /opt/f5/apt-scripts/f5-apt-clean \
    && chown -R ctlr:ctlr "$APPPATH" \
    && rm -f /bin/sh /bin/bash /bin/dash /bin/perl /usr/bin/perl
```

---

## Alternative: Production Build (Non-editable)

For production images, consider using non-editable installs:

```dockerfile
# Build wheels first (in builder stage)
FROM ubuntu:24.04 AS builder
RUN python3 -m venv /app/.venv
COPY src/f5-cccl /tmp/f5-cccl
COPY src/f5-icontrol-rest /tmp/f5-icontrol-rest
COPY src/f5-ctlr-agent /tmp/f5-ctlr-agent

RUN /app/.venv/bin/pip wheel --no-deps --wheel-dir /wheels /tmp/f5-cccl
RUN /app/.venv/bin/pip wheel --no-deps --wheel-dir /wheels /tmp/f5-icontrol-rest
RUN /app/.venv/bin/pip wheel --no-deps --wheel-dir /wheels /tmp/f5-ctlr-agent

# Final stage
FROM ubuntu:24.04
RUN python3 -m venv /app/.venv
COPY --from=builder /wheels /wheels
RUN /app/.venv/bin/pip install --no-cache-dir /wheels/*.whl \
    && rm -rf /wheels
ENV PATH="/app/.venv/bin:$PATH"
```

---

## Testing the Changes

### 1. Local Build Test
```bash
# Build the Docker image
docker build -t f5-bnk-cis:test .

# Verify packages are installed
docker run --rm f5-bnk-cis:test python -c "import f5_cccl.api; print('✓ f5-cccl OK')"
docker run --rm f5-bnk-cis:test python -c "import f5_icontrol_rest; print('✓ f5-icontrol-rest OK')"
```

### 2. Verify setuptools version
```bash
docker run --rm f5-bnk-cis:test pip show setuptools
```

Should show: `Version: 82.0.1`

### 3. Run trivy scan
```bash
trivy image f5-bnk-cis:test --severity HIGH,CRITICAL
```

Should show **0 Python vulnerabilities** for CVE-2026-23949 and CVE-2026-24049.

---

## Troubleshooting

### Issue: "externally-managed-environment"
**Solution**: Use `--break-system-packages` or create a virtual environment (recommended).

### Issue: "No module named 'importlib_resources'"
**Solution**: Add `importlib-resources>=5.0;python_version<"3.9"` to setup_requirements.txt (already added).

### Issue: Schema files not found
**Solution**: Ensure `package_data` is configured in setup.py:
```python
package_data={
    'f5_cccl': ['schemas/*.yaml', 'schemas/*.json', 'schemas/*.yml'],
},
```

### Issue: Build is slower
**Cause**: PEP 517 builds create isolated environments.
**Solution**: Pre-install build dependencies or use build caching.

---

## Migration Checklist

- [ ] Update f5-cccl to use importlib.resources (✅ Done)
- [ ] Update setup_requirements.txt with setuptools 82.0.1 (✅ Done)
- [ ] Modify Dockerfile to use `pip install -e` instead of `setup.py develop`
- [ ] Choose: virtual environment (recommended) or --break-system-packages
- [ ] Update f5-icontrol-rest if it has similar pkg_resources usage
- [ ] Update f5-ctlr-agent if it has similar pkg_resources usage
- [ ] Test Docker build locally
- [ ] Run trivy scan to verify vulnerabilities are fixed
- [ ] Update CI/CD pipeline
- [ ] Document changes in release notes

---

## Related Files

**In f5-cccl repo (✅ Updated):**
- `setup_requirements.txt` - Updated to setuptools 82.0.1, added wheel 0.46.2
- `f5_cccl/api.py` - Migrated from pkg_resources to importlib.resources
- `.trivyignore` - Removed (no longer needed)

**In CIS repo (⚠️ Needs Update):**
- `Dockerfile.ubuntu` - Update pip install commands
- `requirements.txt` - Update to use modern `-e` syntax or local paths
- `Makefile` - Update build commands if needed
- CI/CD configs - Update Docker build steps

---

## References

- setuptools changelog: https://setuptools.pypa.io/en/latest/history.html
- PEP 517 (Build System): https://peps.python.org/pep-0517/
- PEP 668 (Externally Managed): https://peps.python.org/pep-0668/
- importlib.resources: https://docs.python.org/3/library/importlib.resources.html
