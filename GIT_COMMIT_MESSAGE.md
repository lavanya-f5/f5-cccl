# setuptools 82.0.1 Upgrade - Git Commit Message

## Summary

Upgrade to setuptools 82.0.1 to fix CVE-2026-23949 and CVE-2026-24049

## Changes

### Security Fixes
- CVE-2026-23949 (HIGH): jaraco.context path traversal vulnerability
- CVE-2026-24049 (HIGH): wheel privilege escalation vulnerability

### Modified Files

1. **setup_requirements.txt**
   - Upgraded setuptools: 79.0 → 82.0.1
   - Added wheel==0.46.2 (security fix)
   - Added importlib-resources>=5.0 (Python 3.8 compatibility)

2. **f5_cccl/api.py**
   - Migrated from pkg_resources (removed in setuptools 82.0) to importlib.resources
   - Backward compatible with Python 3.8 via importlib-resources backport
   - No functional changes - schema loading works identically

3. **Documentation Added**
   - SETUPTOOLS_82_IMPLEMENTATION.md - Complete implementation summary
   - DOCKERFILE_MIGRATION_GUIDE.md - Instructions for CIS team
   - SETUPTOOLS_UPGRADE_ANALYSIS.md - Security analysis

### Testing
```
✓ f5_cccl.api import successful
✓ setuptools version: 82.0.1
✓ wheel version: 0.46.2
✓ No errors found
```

### Breaking Changes
- setuptools 82.x deprecated `setup.py develop`
- **Action Required**: CIS Dockerfile must be updated to use `pip install -e` (see DOCKERFILE_MIGRATION_GUIDE.md)

### Compatibility
- Python 3.8+: ✅ Fully supported
- Python 3.9+: ✅ Uses built-in importlib.resources
- Python 3.12: ✅ Tested and working

---

## Git Commands

```bash
# Review changes
git status
git diff setup_requirements.txt
git diff f5_cccl/api.py

# Stage changes
git add setup_requirements.txt
git add f5_cccl/api.py
git add SETUPTOOLS_82_IMPLEMENTATION.md
git add DOCKERFILE_MIGRATION_GUIDE.md
git add SETUPTOOLS_UPGRADE_ANALYSIS.md

# Commit
git commit -m "Security: Upgrade to setuptools 82.0.1 to fix CVE-2026-23949 and CVE-2026-24049

- Upgrade setuptools from 79.0 to 82.0.1
- Add wheel 0.46.2 to fix CVE-2026-24049
- Migrate from pkg_resources to importlib.resources (setuptools 82+ compatibility)
- Add importlib-resources for Python 3.8 backward compatibility
- Add comprehensive Dockerfile migration guide for CIS team

Breaking Change: setuptools 82.x requires Docker build updates
See DOCKERFILE_MIGRATION_GUIDE.md for instructions

Fixes: CVE-2026-23949, CVE-2026-24049"

# Push
git push origin <branch-name>
```
