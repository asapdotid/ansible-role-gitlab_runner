<p align="center"> <img src="https://user-images.githubusercontent.com/34257858/129839002-15e3f2c7-3f75-46d4-afae-0fd207d7fdde.png" width="100" height="100"></p>

<h1 align="center">
    Role GitLab Runner
</h1>

<p align="center" style="font-size: 1.2rem;">
    This ansible role install and configure GitLab Runner packages On AlmaLinux, Debian, RedHat and Rocky Linux.
</p>

> GitLab 16.0 and later

This role will install the [official GitLab Runner](https://gitlab.com/gitlab-org/gitlab-runner).
(Custom and inspire from [riemers](https://github.com/riemers/ansible-gitlab-runner)) with updates. Needed something simple and working, this did the trick for me.

### Runner authentication tokens (also called runner tokens)

> In `GitLab 16.0 and later`, you can use an authentication token to register runners instead of a registration token. _Runner registration tokens have been deprecated_.

To generate an authentication token, you create a runner in the GitLab UI and use the authentication token instead of the registration token.

| Process                         | Registration command                                                                                      |
| ------------------------------- | --------------------------------------------------------------------------------------------------------- |
| Registration token (deprecated) | gitlab-runner register --registration-token `$RUNNER_REGISTRATION_TOKEN` _runner configuration arguments_ |
| Authentication token            | gitlab-runner register --token `$RUNNER_AUTHENTICATION_TOKEN`                                             |

[Read Doc](https://docs.gitlab.com/ee/security/token_overview.html)

## Requirements

This role requires Ansible 2.7 or higher.

## Role Variables

| Name                            | Default Value           | Description                                                                       |
| ------------------------------- | ----------------------- | --------------------------------------------------------------------------------- |
| `gitlab_runner_setup`           | `install`               | Specify whether you want to install Gitlab Runner `install`/`upgrade`/`uninstall` |
| `gitlab_runner_version`         | `"16.1.0"`              | The version of the GitLab runner to install                                       |
| `gitlab_runner_coordinator_url` | `"https://gitlab.com/"` | The URL to register the runner                                                    |
| `gitlab_runner_runners`         | `[]`                    | A list of runners to register and configure                                       |

See the [`defaults/main.yml`](https://github.com/asapdotid/ansible-role-gitlab_runner/blob/master/defaults/main.yml) file listing all possible options which you can be passed to a runner registration command.

### Gitlab Runners cache

For each gitlab runner in gitlab_runner_runners you can set cache options. At the moment role support s3, azure and gcs types.
Example configurration for s3 can be:

```yaml
gitlab_runner:
  cache_type: "s3"
  cache_path: "cache"
  cache_shared: true
  cache_s3_server_address: "s3.amazonaws.com"
  cache_s3_access_key: "<access_key>"
  cache_s3_secret_key: "<secret_key>"
  cache_s3_bucket_name: "<bucket_name>"
  cache_s3_bucket_location: "eu-west-1"
  cache_s3_insecure: false
```

### Read Sources

For details follow these links:

- [gitlab-docs/runner: advanced configuration: runners.machine section](https://docs.gitlab.com/runner/configuration/advanced-configuration.html#the-runnersmachine-section)
- [gitlab-docs/runner: autoscale: supported cloud-providers](https://docs.gitlab.com/runner/configuration/autoscale.html#supported-cloud-providers)
- [gitlab-docs/runner: autoscale_aws: runners.machine section](https://docs.gitlab.com/runner/configuration/runner_autoscale_aws/#the-runnersmachine-section)

## Example Playbook

```yaml
- hosts: all
  become: true
  vars_files:
    - vars/main.yml
  roles:
    - { role: asapdotid.gitlab_runner }
```

Inside `vars/main.yml`

```yaml
gitlab_runner_runners:
  # Using docker executor
  - name: Gitlab-Runner Docker
    state: present
    token: "{{ vault_gitlab_runner_docker_token }}"
    # url is an optional override to the global gitlab_runner_coordinator_url
    url: https://gitlab.com
    executor: docker
    docker_image: alpine
    docker_volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "/cache"
    extra_configs:
      runners.docker:
        memory: 512m
        allowed_images:
          - node:*
          - alpine:*
          - asapdotid/node-alpine:*
          - asapdotid/ansible-alpine:*
      runners.docker.sysctls:
        net.ipv4.ip_forward: "1"
    cache_type: s3
    cache_path: app-cache
    cache_shared: false
    cache_s3_server_address: "{{ __minio_host }}:{{ __minio_port }}"
    cache_s3_access_key: "{{ __minio_root_user }}"
    cache_s3_secret_key: "{{ __minio_root_password }}"
    cache_s3_bucket_name: "{{ __minio_default_bucket }}"
    cache_s3_insecure: true
    # Using shell executor
  - name: Gitlab-Runner Shell
    state: present
    token: "{{ vault_gitlab_runner_shell_token }}"
    # url is an optional override to the global gitlab_runner_coordinator_url
    url: https://gitlab.com
    executor: shell
```

### Autoscale setup on AWS

how `vars/main.yml` would look like, if you setup an autoscaling GitLab-Runner on AWS:

```yaml
gitlab_runner_runners:
  - name: "Example autoscaling GitLab Runner"
    state: present
    # token is an optional override to the global gitlab_runner_registration_token
    token: "{{ vault_gitlab_runner_docker_machine_token }}"
    executor: "docker+machine"
    docker_image: "alpine"
    # Indicates whether this runner can pick jobs without tags.
    run_untagged: true
    extra_configs:
      runners.machine:
        IdleCount: 1
        IdleTime: 1800
        MaxBuilds: 10
        MachineDriver: "amazonec2"
        MachineName: "git-runner-%s"
        MachineOptions:
          [
            "amazonec2-access-key={{ lookup('env','AWS_IAM_ACCESS_KEY') }}",
            "amazonec2-secret-key={{ lookup('env','AWS_IAM_SECRET_KEY') }}",
            "amazonec2-zone={{ lookup('env','AWS_EC2_ZONE') }}",
            "amazonec2-region={{ lookup('env','AWS_EC2_REGION') }}",
            "amazonec2-vpc-id={{ lookup('env','AWS_VPC_ID') }}",
            "amazonec2-subnet-id={{ lookup('env','AWS_SUBNET_ID') }}",
            "amazonec2-use-private-address=true",
            "amazonec2-tags=gitlab-runner",
            "amazonec2-security-group={{ lookup('env','AWS_EC2_SECURITY_GROUP') }}",
            "amazonec2-instance-type={{ lookup('env','AWS_EC2_INSTANCE_TYPE') }}",
          ]
```

#### NOTE

from https://docs.gitlab.com/runner/executors/docker_machine.html:

> The **first time** you’re using Docker Machine, it’s best to execute **manually** `docker-machine create...` with your chosen driver and **all options from the MachineOptions** section. This will set up the Docker Machine environment properly and will also be a good validation of the specified options. After this, you _can destroy the machine_ with `docker-machine rm [machine_name]` and start the Runner.

Example:

```bash
docker-machine create -d amazonec2 --amazonec2-zone=a --amazonec2-region=us-east-1 --amazonec2-vpc-id=vpc-11111111 --amazonec2-subnet-id=subnet-1111111 --amazonec2-use-private-address=true --amazonec2-tags=gitlab-runner --amazonec2-instance-type=t3.medium test
```

Remove docker machine:

```bash
docker-machine rm test
```

## Dependencies

--

## License

Apache License V2

## Author Information

[JogjaScript](https://jogjascript.com)

This role was created in 2023 by [Asapdotid](https://github.com/asapdotid).
