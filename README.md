# GitLab

Docker-Compose based deployment of an on-premise GitLab instance.

## Table of Contents

- [GitLab](#gitlab)
  - [Table of Contents](#table-of-contents)
  - [Background](#background)
    - [Quations and Answers](#quations-and-answers)
  - [Requirements](#requirements)
  - [Preparations](#preparations)
  - [Deployment](#deployment)
  - [Usage](#usage)

## Background

### Quations and Answers

1. Q: Why should we have two different uniqe static IPv4 addresses? <br>
   A: Git technology uses the Secure Shell (SSH) protocol which provides a secure channel over an unsecured network.
   By default, this protocol runs over the 22 port. In a standard Linux server, this port is already taken by the Secure Shell (SSH) daemon. <br>
   It is possiable to use a different port for the Git server requirements, but I find it better to mount the 22 port of the GitLab server container to a dedicated IPv4 address of its own.

2. Q: Which operating systems are supported? <br>
   A: This deployment process was prefectly tested on an Ubuntu 22.04 LTS (Jammy Jellyfish) virtual machine (ESXi). <br> Other Linux-based operating systems might be supported as well.

## Requirements

1. A dedicated Linux server.
   - [Recommended system resources](http://mmb.irbbarcelona.org/gitlab/help/install/requirements.md).
   - Networking:
     - An uniqe static IPv4 address for the Secure Shell mangmanet needs.
     - An uniqe static IPv4 address for the GitLab server itself.
   - Installed Packages:
     - Docker.
     - Docker Compose.

2. A uniqe internal domain name with Secure Socket Layer (SSL) self-signed certificates.
   - The Secure Socket Layer (SSL) certificate of the Certificate Authority (CA) that signed the uniqe internal domain certificates.
   - The self-signed certificate and private key for the uniqe internal domain.

## Preparations

1. Assign the server two uniqe static IPv4 addresses by editing the netowrking configuration file:

    ```bash
    sudo vim /etc/netplan/00-installer-config.yaml
    ```

    ```yaml
    network:
    version: 2
    renderer: "networkd"
    ethernets:
        ens160:
        addresses:
            - "< Secure Shell IPv4 address >/24"
            - "< GitLab server IPv4 address >/24"
        nameservers:
            addresses:
            - "1.1.1.1"
            - "8.8.8.8"
            search: []
        routes:
            - to: "default"
            via: "< Default IPv4 Gateway >"
    ```

2. Apply the networking chanages:

    ```bash
    sudo netplan apply
    ```

3. Validate that the Secure Sheel (SSL) daemon is listening only on one address by adding the following line to its configuration file:

    ```bash
    sudo vim /etc/ssh/sshd_config
    ```

    ```txt
    ...
    ListenAddress < Secure Shell IPv4 address >
    ...
    ```

4. Restart the Secoure Shell (SSL) daemon:

    ```bash
    systemctl restart sshd.service
    ```

5. Add your internal domain name to the end of your hosts file:

    ```bash
    sudo vim /etc/hosts
    ```

    ```bash
    ...
    gitlab.< domain name > < GitLab server IPv4 address >
    ...
    ```

## Deployment

1. Clone this Git repository to the home directory of your dedicated server:

    ```bash
    git clone https://github.com/ArielLahiany/gitlab.git
    ```

2. Inside the cloned directory, create the Secure Sockets Layer (SSL) directory:
    ```bash
    mkdir ssl
    ```

3. Inside the cloned directory, create the gitlab-runner directory:
   ```bash
   mkdir gitlab-runner
   ```

4. Copy the Secure Sockets Layer (SSL) certificates to the created directory.

5. Validate your directories tree structure:

    ```bash
    ├── .gitignore
    ├── README.md
    ├── docker-compose.yaml
    ├── gitlab-runner
    └── ssl
        ├── ca.crt
        ├── gitlab.< domain name >.crt
        └── gitlab.< domain name >.key
    ```

6. Edit the docker-compose.yaml file to your requirements.

7. Pull the public Docker images:
   ```bash
   docker-compose pull
   ```

8. Spin up the stack:
   ```bash
   docker-compose up -d
   ```

## Usage

1. Get the random-generated root password:
    ```bash
    sudo cat /var/lib/docker/volumes/gitlab-server-config/_data/initial_root_password
    ```

    ```txt
    ...
    Password: < random-generated >
    ...
    ```

2. Browse to the GitLab server IPv4 address.

3. Log in to the system by using the default username and password:
   ```txt
   username: root
   password: < random-genereted >
   ```

3. Navigate to the runners section of the admin panel:
    ```txt
    https://gitlab.< domain name >/admin/runners
    ```

4. Get an uniqe registertion token.

5. Copy the Certificate Authority (CA) certificate to the temporary directory:
   ```bash
   cp ssl/ca.crt /tmp
   ```

6. Register the GitLab Runner instance:
   ```bash
   docker run --rm -it \
   -v /home/ubuntu/gitlab/gitlab-runner:/etc/gitlab-runner \
   -v /tmp/ca.crt:/etc/gitlab-runner/certs/ca.crt:ro \
   --add-host=gitlab.< domain name >:< GitLab server IPv4 address > \
   gitlab/gitlab-runner:alpine3.15-v15.2.2 register \
   --tls-ca-file /etc/gitlab-runner/certs/ca.crt
   ```

7. Edit the created GitLab runner configuration file:
   ```bash
   vim gitlab-runner/config.toml
   ```

    ```toml
    concurrent = 4
    check_interval = 0
    log_level = "info"

    [session_server]
    session_timeout = 1800

    [[runners]]
    name = "< GitLab runner name >"
    url = "https://gitlab.< domain name >"
    token = "< uniqe registration token >"
    tls-ca-file = "/etc/gitlab-runner/certs/ca.crt"
    executor = "docker"
    [runners.docker]
        tls_verify = false
        image = "docker:20.10.18-dind"
        privileged = true
        network_mode = "gitlab"
        disable_entrypoint_overwrite = false
        oom_kill_disable = false
        disable_cache = false
        volumes = [
            "/cache",
            "/etc/docker/certs.d:/etc/docker/certs.d:ro"
        ]
        shm_size = 0
    ```
