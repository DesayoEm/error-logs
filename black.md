**Date:** 19 Jan 2025  
**Context:** Setting up GitHub Actions CI workflow to run Black formatting check. Black passed locally on Windows but failed in CI on Ubuntu.  
**Version:** Black 24.4.2, Python 3.11, GitHub Actions ubuntu-latest

## Error
```
would reformat /home/runner/work/.../adverse_events.py
would reformat /home/runner/work/.../references.py
would reformat /home/runner/work/.../failure_generators.py
would reformat /home/runner/work/.../study_transformation.py

Oh no! 💥 💔 💥
4 files would be reformatted, 65 files would be left unchanged.
Error: Process completed with exit code 1.
```

## Code (Before)
```yaml
- name: Install linting tools
  run: pip install black

- name: Run black (formatting check)
  run: black --check .
```

## Solution
```yaml
- name: Install linting tools
  run: pip install black==24.4.2  # pin to the same version as local

- name: Run black (formatting check)
  run: black --check .
```

## Explanation

- **What caused the error** - The CI installed the latest version of Black from PyPI, which may have different formatting rules than the version installed locally. Different Black versions can make different formatting decisions, so code that passes locally can fail in CI.

- **Why the solution works** - Pinning Black to the exact version used locally ensures identical formatting rules in both environments. Run `black --version` locally to find your version.


