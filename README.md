# python-template-uv

## Introduction

**This repo is originally from my local CI/CD environment using forgejo, this one you're viewing is the mirror version in GitHub.**

Default Python Project Template (using uv)

It is a default python project template using uv for new repository, please do the following steps after creating a new repository

1. Update pyproject.toml
   - name, version, description under project section
   - dependencies
   - project scripts
   - Comment tool.uv.index and tool.uv.sources if private nexus is not required 
2. Rename the package name (python_template_uv) under src
3. Update REAMDE.md (this file)
4. Remove Python files under src and tests as they are dummy file.

### 

## Notice for local development when private nexus is required.

Please create a .netrc in the User home directory (e.g. ~/.netrc)

```
skl@saturn:~$ cat .netrc 
machine saturn
login skl
password supersecret123456
```

```
uv sync --system-certs
```