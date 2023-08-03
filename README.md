<p align="center"> <img src="https://user-images.githubusercontent.com/34257858/129839002-15e3f2c7-3f75-46d4-afae-0fd207d7fdde.png" width="100" height="100"></p>

<h1 align="center">
    Role GitLab Runner
</h1>

<p align="center" style="font-size: 1.2rem;">
    This ansible role install and configure GitLab Runner packages On Ubuntu, CentOS.
</p>

> GitLab 16.0 and later

This role will install the [official GitLab Runner](https://gitlab.com/gitlab-org/gitlab-runner)
(custom and inspire from [riemers](https://github.com/riemers/ansible-gitlab-runner)) with updates. Needed something simple and working, this did the trick for me.

### Runner authentication tokens (also called runner tokens)

In `GitLab 16.0 and later`, you can use an authentication token to register runners instead of a registration token. Runner registration tokens have been deprecated.

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
  - name: "Example Docker GitLab Runner"
    # GitLab runner token
    token: "abcd"
    # url is an optional override to the global gitlab_runner_coordinator_url
    url: "https://gitlab.com"
    executor: docker
    docker_image: "alpine"
    tags:
      - node
      - ruby
      - mysql
    docker_volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "/cache"
    docker_services:
      - name: "redis:2.8"
        alias: "cache"
    extra_configs:
      runners.docker:
        memory: 512m
        allowed_images: ["ruby:*", "python:*", "php:*"]
      runners.docker.sysctls:
        net.ipv4.ip_forward: "1"
```

## Dependencies

--

## License

Apache License V2

## Author Information

[JogjaScript](https://jogjascript.com)

This role was created in 2023 by [Asapdotid](https://github.com/asapdotid).
