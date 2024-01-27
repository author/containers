# Setup Script

This Docker image helps acquire content from Github repositories (both public and private). It provides a quick and secure way to retrieve infrastructure assets without having to configure git, SSH, FTP, or other utilities. The intention is to help rapidly bootstrap common infrastructure in a Docker environment with a short and memorable command.

The command assumes a source repository contains setup scripts and assets in one of two arrangements: Directory-Based or Branch-Based.

**Directory-Based Configurations**
The target repository must be arranged with each top level directory representing a configuration.

```sh
myorg/repo
├── discovery
│   ├── docker-compose.yml
│   └── setup.sh
├── ldap
│   ├── docker-compose.yml
│   └── setup.sh
├── web
│   ├── docker-compose.yml
│   └── setup.sh
├── api
│   ├── docker-compose.yml
│   └── setup.sh
└── database
    ├── docker-compose.yml
    └── setup.sh
```

**Branch-Based Configuration**
In this arrangement, each branch represents a configuration. Source files are assumed to exist within the branch.

```sh
myorg/repo branches
├── main
├── discovery
├── ldap
├── web
├── api
└── database
```

## Memorable Command

```sh
docker run --rm -it \
  -e "GITHUB_USER=github_username" \
  -e "GITHUB_TOKEN=github_personal_access_token" \
  -e "GITHUB_ORG=organization" \ # optional
  -v "./:/output" \
  author/setup
```

This command has two required environment variables (Github user and token) and one optional environment variable (Github organization). The volume is also required. The mounted volume is where Github files are downloaded to on the host. In this example, we use the relative `./` directory, which will load the files into the current working directory.

### Usage

This command automatically pulls the container from Docker Hub and runs a simple prompt wizard. If you specfiy a Github Organization, a list of repositories will be presented to choose from.

```sh
Available repositories:
  1. myorg/discovery
  2. myorg/ldap
  3. myorg/web
  4. myorg/api
  5. myorg/database

Enter the number of the repo you wish to setup:
```

If no organization is specified, you will be prompted to manually input the org/repository you wish to connect to.

Next, the wizard prompts to determine whether you want to use the `Branch` or `Directory` method for retrieving content.

```sh
Using author/infrastructure

Setup from:
  1. Branch
  2. Directory
  q. Quit

Enter the number of the repo arrangement:
```

If `Branch` is selected, a prompt will appear to select the branch:

```sh
Select a branch:
  1. infrastructure-dev
  2. infrastructure-prod
  3. main
  q. Quit

Enter the number of your choice:
```

If `Directory` is selected, the top level directories of the repository will be listed, allowing selection of the directory to download.

```sh
Retrieving myorg/containers top level directories...

Select a directory or Quit:
  1. auto-letsencrypt-cloudflare
  2. docker-host-ipc
  q. Quit

Enter the number of your choice:
```

Once a branch or directory is selected, the wizard downloads the contents to the container's `/app` directory (which is mapped to the host container).

When complete, the container will exit and remove itself.

**Tip*
If you wish to remove the image after the wizard is complete, add `&& docker rmi author/setup` to the end of the command. For example:

```sh
docker run --rm -it \
  -e "GITHUB_USER=github_username" \
  -e "GITHUB_TOKEN=github_personal_access_token" \
  -e "GITHUB_ORG=organization" \ # optional
  -v "./:/output" \
  author/setup \
&& docker rmi author/setup
```

We recommend doing this the last time you plan to run the command. The image is only relevant to setup and is unncessary once setup is complete.

### Next Steps

It is assumed some sort of scripts are now available. In our practice, we create setup wizards for each type of infrastructure component we deploy. These are stored in a file called `setup`. This way, no matter which server we're configuring, there is a common file to setup/install what we need. Sometimes this script is as simple as `docker compose up -d`. Other times we have a wizard that generates a Dockerfile from a series of questions, or commands to install dependencies.

As a result, we use the following command and attempt to commit it to memory:

```sh
docker run --rm -it \
  -e "GITHUB_USER=github_username" \
  -e "GITHUB_TOKEN=github_personal_access_token" \
  -e "GITHUB_ORG=organization" \
  -v "./:/output" \
  author/setup \
&& docker rmi author/setup \
&& chmod +x ./setup \
&& ./setup
```

This command downloads setup files, prepares permissions, and executes the `setup` file. As a result, administrators are greeted with a series of easy-to-answer prompts that automate the setup of our environments.

## Why?

There are situations where the overhead of tools like Kubernetes (and even K3S) don't justify the extra effort. It can be easier to maintain configurations and updates in a single git repository as opposed to needing a database/consul/etcd. Bottom line: sometimes the simplest solution is just a series of scripts.
