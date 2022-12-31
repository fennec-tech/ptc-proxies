# PTC Proxies
This project provides a Docker compose environment to run Pull-Through Cache (PTC) proxy servers, and allow caching downloads for package/image management tools such as docker, pip, apt, and npm.

---
## Overview
### Features
- `Docker` registry using the official [Registry](https://hub.docker.com/_/registry) image.
- `apt` proxy using a [apt-cacher-ng](https://help.ubuntu.com/community/Apt-Cacher%20NG) image from [deployable](https://hub.docker.com/r/deployable/apt-cacher-ng) server.
- `pip` index using a docker [pypicloud](https://pypicloud.readthedocs.io/en/latest/) image by [stevarc](https://hub.docker.com/r/stevearc/pypicloud).
- `npm` registry powered by [verdaccio](https://verdaccio.org/) using [their official docker image](https://hub.docker.com/r/verdaccio/verdaccio)

---
## Server Configuration
### 1. Requirements
This has been tested so far on Ubuntu, so it probably works on Linux. I can't guarantee the same for other operating systems.
All you need to make this work is:
- Docker
- Docker compose

### 2. Install
Clone or download the zipped repository files
```
git clone https://github.com/fennec-tech/ptc-proxies
```

### 3. Configuration
You can edit `.env` file to change the ports defined for each service.

 The default ports are as follow:
- Docker (Registry) : **7760**
- apt (AptCacherNg) : **7761**
- pip (Pypicloud) : **7762**
- npm (Verdaccio): **7763**

You can also browse the `config` directory to find the configuration files mounted at each container's startup:
- Docker registry: `./config/docker/config.yml`
- AptCacherNg: `./config/apt/acng.conf`
- PypiCloud: `./config/pip/config.ini`
- Verdaccio: `./config/npm/acng.conf`

### 4. Launch
At the project's root, run the following command:
```
docker-compose up -d
```
### 5. Troubleshooting
Sometimes, maximum memory or disk space are reached, resulting in the services failing to download, cache, or process requests. If that's the case you can try removing all containers, then recreating them:
```
docker-compose down
docker-compose up -d
```

If the problem persists, then try rebooting your machine.
If it's still unsolved, then you can contact us and file a [GitHub issue](https://github.com/fennec-tech/ptc-proxies/issues)

### 6. Volume permissions
In order for our proxies to cache files, permissions, need to be adjusted. After starting containers, please run the following commands.
```
# for pypicloud (pip)
docker exec -it ptc-pip-1 bash -c 'chown -R pypicloud:pypicloud /var/lib/pypicloud/packages'

# for verdaccio (npm)
docker exec -it ptc-npm-1 bash -c 'chown -R verdaccio:verdaccio /verdaccio/storage'
```

---
## Client Configuration

### 1. Docker registry
To use the registry as a PTC globally on your host machine, create or update the file `/etc/docker/daemon.json`
and add the following line to it:
```
{
    "registry-mirrors": ["http://127.0.0.1:7760"]
}
```

Then restart docker daemon by running the command:
```
sudo systemctl restart docker
```

### 2. apt cache proxy
**a. Use proxy globally**

Create a new file at `/etc/apt/apt.conf.d/00proxy` and add the following line to it:
```
Acquire::http::Proxy "http://127.0.0.1:7761";
```

**b. Setup proxy using environment variable**
You can define an environment variable to use apt proxy.
```
#!/bin/sh

# For HTTP connection (useful for local access)
export HTTP_PROXY="http://127.0.0.1:7761"

# For HTTPS connection to provide secure remote access
export HTTP_PROXY="https://<your-proxy-address>:<proxy-port>"
```

Please note these variables are also detected by `pip`.
So if you plan to use `pip install` during the same session, then you should unset the variable first. Otherwise, all `pip` traffic will be redirected to your apt-cache-ng server then fail to install.

To unset the variable:
```
#!/bin/sh

# For HTTP connections
unset HTTP_PROXY

# For HTTPS connections 
unset HTTPS_PROXY
```

### 3. pip index
pip supports either redefining the default index url, or adding an extra index url.
But if you want to always go through your pip proxy, then you should set the index url.

**a. configure globally**

Run one of the following bash commands options:
```
# Option 1: change the default index url
pip config set global.index-url http://127.0.0.1:7762/simple

# Option 2: extra index url
pip config set global.extra-index-url http://127.0.0.1:7762/simple
```

If you are using a domain name `<your-domain>` instead of 127.0.0.1, then you should configure it as trusted
```
pip config set global.trusted-host "pypi.org file.pythonhosted.org ptc.local:7762"
```

Those configurations are saved to `<your-user-home>/.config/pip/pip.conf`

**b. configure per environment**

You can use the environment variables recognized by pip:
```
# Option 1: change the default index url
export PIP_INDEX_URL=http://127.0.0.1:7762/simple

# Option 2: extra index url
export PIP_INDEX_URL=http://127.0.0.1:7762/simple
```

If you are using a domain name `<your-domain>` instead of 127.0.0.1, then you should configure it as trusted
```
export PIP_TRUSTED_HOST="pypi.org file.pythonhosted.org <your-domain>"
```

**d. use per command**

You can directly specify with `pip install` additional options.
```
# Option 1: change the default index url
pip install --index-url=http://127.0.0.1:7762/simple ...

# Option 2: extra index url
pip install --extra-index-url=http://127.0.0.1:7762/simple ...
```

If you are using a domain name `<your-domain>` instead of 127.0.0.1, then you should configure it as trusted

```
pip install [your index configuration here] --trusted-host="pypi.org file.pythonhosted.org <your-domain>" ...
```

### NPM registry
**a. configure globally**
Run the following bash command
```
npm config -g set registry=http://localhost:4940
```
**b. configure per user**
Run the following shell command
```
npm config -g set registry=http://localhost:4940
```

**d. configure per project**
edit or create `.npmrc` file at your project's root directory, to add the following line:
```
registry=http://localhost:4940
```

**c. configure per project**
Add the `--registry` option to your npm install command
```
npm install --registry=http://localhost:4940 ...
```

## Contribution
You can simply fork this project and do your customizations. Then if you think it's worth contributing back, you can open an [pull request](https://github.com/fennec-tech/ptc-proxies/pulls) for it.

If you need help using `ptc-proxies`, or just found a bug, please reach out by filing a [GitHub issue](https://github.com/fennec-tech/ptc-proxies/issues)

## Licensing
This project is licensed under [MIT Open License](LICENSE).