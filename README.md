# python-template-uv

Default Python Project Template (using uv)


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