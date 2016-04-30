### Directory structure

App must be structured like this:

```
app/
build/
bin/
uberspace.config
```

It's recommended to exclude build/ from your git repository via .gitignore


### Setup

1. Move your app to match the structure
2. Edit userspace.config to match your setup
3. Add a SSL key to your uberspace account, so the script can login automatically. Try 'ssh username@username.subdomain.uberspace.de', it should log you in directly, without asking for password or username

### Deployment

1. cd to your app main directory, where uberspace.config is
2. run bin/deploy-to-uberspace
