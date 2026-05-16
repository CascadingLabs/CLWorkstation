---
name: verification-plan
description: Use when deciding how to validate a CLWorkstation change with dry-runs, molecule, or targeted role checks.
---
# CLWorkstation Verification Plan
Prefer targeted `ansible-playbook ... --check --diff` when useful, and `molecule test` for behavior changes that need container verification across supported distros.
