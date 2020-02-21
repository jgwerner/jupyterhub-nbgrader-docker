# JupyterHub + nbgrader + docker-compose

## Status

**Proof of Concept (POC)**

## Prerequisites

On remote host:

- Ubuntu 18.04

On machine running `ansible-playbook`:

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) >= 2.8

## Quick Start

1. Create a `ansible/hosts` file from the provided `ansible/hosts.example`:

    cp `ansible/hosts.example` `ansible/hosts`

2. Update `private_key_file` in the `hosts` file with the full path to the PEM key to access your instance.

3. Update `ansible/ansible.cfg` with your IPv4 address.

4. Run ansible playbook:

In it's basic form:

```bash
ansible-playbook \
  provisioning.yml
```

With custom variables and with verbose output:

```bash
ansible-playbook \
  provisioning.yml \
  --extra-vars \
  "org_name=myedu"
  -v
```

All global variable names are listed in `ansible/group_vars/all.yml`.

## Components

* **JupyterHub**: Runs [JupyterHub](https://jupyterhub.readthedocs.org/en/latest/getting-started.html#overview) within a Docker container running as root.

* **Authenticator**: Authentication service. For development, this setup uses a customized version of either the [FirstUseAuthenticator](https://github.com/jupyterhub/dummyauthenticator).

* **Spawner**: Spawning service to manage user notebooks. This setup uses a custom [DockerSpawner](https://github.com/jupyterhub/dockerspawner).

* **Data Directories**: Data directories for configuration files, databases, etc rely on the containers themselves or by mounting directories from the host.

* **Databases**: This setup relies on the default `SQLite` databases for both JupyterHub and nbgrader.

* **Network**: An external bridge network named `jupyter-network` is used by default. The grader notebook service and the end-user notebooks are attached to this network when spawned.

## Customization

### Configuration Files

The configuration changes depending on how you decide to update this setup. Essentially customizations boil down to:

1. JupyterHub configuration using `jupyterhub_config.py`:

    - Authenticators
    - Spawners
    - Services

> **Note**: By default the `jupyterhub_config.py` file is located in `/etc/jupyter/jupyterhub_config.py` within the running JupyterHub container, however, if you change this location (which would require an update to the JupyterHub's Dockerfile) then you need to make sure you are using the correct configuration file with the `jupyterhub -f /path/to/jupyterhub_config.py` option.

Whenever possible we try to adhere to [JupyterHub's](https://jupyterhub.readthedocs.io/en/stable/installation-basics.html#folders-and-file-locations) recommended paths:

- `/srv/jupyterhub` for all security and runtime files
- `/etc/jupyterhub` for all configuration files
- `/var/log` for log files

2. Nbgrader configurations using `nbgrader_config.py`.

Three `nbgrader_config.py` files should exist, two for the shared grader account and one for each instructor/learner account:

**Grader Account**

* **Grader's home: `/home/grader-{course_id}/.jupyter/nbgrader_config.py`**: defines how `nbgrader` authenticates with a third party service, such as `JupyterHub` using the `JupyterHubAuthPlugin`, the log file's location, and the `course_id` the grader account manages.
* **Grader's course: `/home/grader-{course_id}/{course_id}/nbgrader_config.py`**: configurations related to how the course files themselves are managed, such as solution delimeters, code stubs, etc.

**Instructor/Learner Account**

* **Instructor/Learner settings `/etc/jupyterhub/nbgrader_config.py`**: defines how `nbgrader` authenticates with a third party service, such as `JupyterHub` using the `JupyterHubAuthPlugin`, the log file's location, etc. Instructor and learner accounts do **NOT** contain the `course_id` identifier in their nbgrader configuration files.

> **Note**: nbgrader utilizes the directory structure within the exchange root (for example `/srv/nbgrader/exchange`) to list courses the instructors and learners have access to. If you don't see a particular course listed in the instructor's course list extension then make sure the course directory exists and that it has the right permissios.

3. Jupyter Notebook configuration using `jupyter_notebook_config.py`. This configuration is standard fare and unless required does not need customized edits.

4. For this setup, the deployment configuration is defined primarily with `docker-compose.yml`.

### Build the Stack

The following docker images are created with the playbook:

- Jupyter Notebook Student image
- Jupyter Notebook Instructor image
- Jupyter Notebook shared Grader image
- JupyterHub image
- JupyterHub configurable-http-proxy image (pulled)

When building the images the configuration files are copied to the image from the host using the `COPY` command. Environment variables are stored in `env.*` files. You can either customize the environment variables within the `env.*` files or add new ones as needed. The `env.*` files are used by docker-compose to reduce the file's verbosity.

### Authenticator

To change the `authenticator` used by JupyterHub, edit the `JupyterHub.authenticator_class` setting in `jupyterhub_config.py` to use your desired authenticator. For example, to change authenticator from `DummyAuthenticator` to `FirstUseAuthenticator`:

Before:

```python
c.JupyterHub.authenticator_class = 'dummyauthenticator.DummyAuthenticator'
```

After:

```python
c.JupyterHub.authenticator_class = 'firstuseauthenticator.FirstUseAuthenticator'
```

For local dev testing, we recommend the [DummyAuthenticator](https://github.com/jupyterhub/dummyauthenticator) since it provides a simple way to log in without having to worry about setting password policies and the like.

Consult the authenticator's official documentation to ensure that all configurations are set before launching JupyterHub.

### Spawner

By default this setup includes the `CustomDockerSpawner` class which extends the `DockerSpawner` class. This implementation calls the `authenticator` function to get check for the user's group membership and uses the `pre_spawn_hook` to set the user's image based on user role.

Edit the `JupyterHub.spawner_class` to update the spawner used by JupyterHub when launching user containers. For example, if you are changing the spawner from `DockerSpawner` to `KubeSpawner`:

Before:

```python
c.JupyterHub.spawner_class = 'dockerspawner.DockerSpawner'
```

After:

```python
c.JupyterHub.spawner_class = 'kubespawner.KubeSpawner'
```

As mentioned in the [authenticator](#authenticator) section, make sure you refer to the spawner's documentation to consider all settings before launching JupyterHub.

### Proxies

Users connect to the proxy, not directly to JupyterHub. JupyterHub updates the proxies' routing tables using an internal facing port (`8001` by default). Users connect to the proxy using an external facing port (`8000` by default).

This setup use JupyterHub's [configurable-http-proxy]((https://github.com/jupyterhub/configurable-http-proxy)) running in a **separate container** which enables JupyterHub restarts without interrupting active sessions between end-users and their Jupyter Notebooks. For the sake of simplicity, this setup does not include TSL termination at the proxy.

### Jupyter Notebook Images

**Requirements**

- The Jupyter Notebook image needs to have `JupyterHub` installed and this version of JupyterHub **must coincide with the version of JupyterHub that is spawing the Jupyter Notebook**. By default the `jupyter/docker-stacks` images have JupyterHub installed.
- Use one of images provided by the [`jupyter/docker-stacks`](https://github.com/jupyter/docker-stacks) images as the base image.
- Make sure the image is on the host used by the spawner to launch the user's Jupyter Notebook.

There are four notebook images:

- [Base](roles/common/templates/Dockerfile.base.j2)
- [Student](roles/common/templates/Dockerfile.student.j2)
- [Learner](roles/common/templates/Dockerfile.learner.j2)
- [Grader](roles/common/templates/Dockerfile.grader.j2)

The nbgrader extensions are enabled within the images like so:

|   | Students  | Instructors  | Formgraders  |
|---|---|---|---|
| Create Assignment  | no  | no  | yes  |
| Assignment List  | yes  | yes | no  |
| Formgrader  | no  | no  | yes  |
| Course List | no  | yes  | no  |

Refer to [this section](https://nbgrader.readthedocs.io/en/stable/user_guide/installation.html#installing-and-activating-extensions) of the `nbgrader` docs for more information on how you can enable and disable specific extensions.

### Grading with Multiple Instructors

As of `nbgrader 0.6.0`, nbgrader supports the [JupyterHubAuthPlugin](https://nbgrader.readthedocs.io/en/stable/configuration/jupyterhub_config.html#jupyterhub-authentication) to determine the user's membership within a course. The section that describes how to run [nbgrader with JupyterHub] is well written. However for the sake of clarity, some of the key points and examples are written below.

The following rules are defined to determine access to nbgrader features:

- Users with the student role are members of the `nbgrader-{course_id}` group(s). Students are shown assignments only for course(s) with `{course_id}`.
- Users with the instructor role are members of the `formgrade-{course_id}` group(s). Instructors are shown links to course(s) to access `{course_id}`. To access the formgrader, instructors access to the `{course_id}` service (essentially a shared notebook) and authenticate to the `{course_id}` service using JupyterHub as an OAuth2 server.

> **NOTE** It's important to emphasize that with this setup **instructors do not grade assignments with their own notebook server** but with a **shared notebook** which runs as a JupyterHub service and which is owned by the shared `grader-{course_id}` account.

The configuration for this setup requires one `jupyterhub_config.py` and three `nbgrader_config.py`'s.

1. Within `jupyterhub_config.py` which defines the shared grader service:

- Name
- Access by group
- Ownership
- API token
- URL
- Command

For example for a course named `intro101`:

```python
c.JupyterHub.services = [
    {
        'name': 'intro101',
        'url': 'http://intro101:8888',
        'oauth_no_confirm': True,
        'admin': True,
        'api_token': 'my_secure_token',
    },
]
```

2. The global `nbgrader_config.py` used by all roles, located in `/etc/jupyterhub/nbgrader_config.py` which defines:

- Authenticator plugin class
- Exchange directory location

For example:

```python
c.Exchange.path_includes_course = True
c.Exchange.root = '/srv/nbgrader/exchange'
c.Authenticator.plugin_class = JupyterHubAuthPlugin
```

3. The `nbgrader_config.py` located within the shared grader account home directory: (`/home/grader-{course_id}/.jupyter/nbgrader_config.py`) which defines:

- Course root path
- Course name

For example:

```python
c.CourseDirectory.root = '/home/grader-intro101/intro101'
c.CourseDirectory.course_id = 'intro101'
```

4. The `nbgrader_config.py` located within the course directory: (`/home/grader-{course_id}/{course_id}/nbgrader_config.py`) which defines:

- The course_id
- Nbgrader application options

For example:

```python
c.CourseDirectory.course_id = 'intro101'
c.ClearSolutions.text_stub = 'ADD YOUR ANSWER HERE'
```

### Some Notes on Authentication, User Directories, and Local System Users

Whether or not the `Authenticator` and `Spawner` require a local system user can be a source of confusion. JupyterHub's default authenticators, [LocalAuthenticator](https://github.com/jupyterhub/jupyterhub/blob/master/jupyterhub/auth.py#L613) and [PAMAuthenticator](https://github.com/jupyterhub/jupyterhub/blob/master/jupyterhub/auth.py#L789), require local system users.

This setup assumes that users are **not** local system users. Custom authentication classes should extend the base `Authenticator` class and override the `authenticate` method. Further customizations can be provided by overriding the `normalize_username` and `check_whitelist` methods.

Since users require their **own directories to manage their files and folders**, additional steps need to take place to create these directories without having to create local system users. In our opinion the best approach is to define a method to create user directories and assign this method to the `Spawner.pre_spawn_hook` located within `jupyterhub_config.py` to accomplish this task.

## Gotchas

- **Permissions**: student (i.e. Learner role) should not have access to the grader's directories, as that would give them access to the sources. Grader directories are assignment a different `NB_UID` (10001 by default) but rely on the same standard `NB_GID`.
- **Order matters for docker volume mounts**: you need to create the grader's home directory before mounting the docker volume. Otherwise, the configs in the docker container's volume take precedence over the host directory configs. Nevertheless, if the docker mount creates the directory then said directory would have root:root for uid/gid, which would result in errors.
- **JupyterHub pre-flight API token(s)**: setting a pre-flight token `JupyterHub.service_tokens` removes the need for launching the stack, obtaining a token for a JupyterHub user, adding it to `jupyterhub_config.py` and restarting the JupyterHub service.
- **Notebook images tagged by user role**: this setup creates an image for each user role instead of using a script to enable/disable the nbgrader extensions by appending or replacing the `singleuser-notebook.sh` script. The pros are better start times and it avoids possible race conditions. Cons are that setting up the authenticator(s) to obtain the `USER_ROLE` requires some extra setup.
- **Creating user directories**: system users are not created with the `DockerSpawner`, however all users need their own directories. Therefore this setup uses the `Spawner.pre_spawn_hook` within the customized `DockerSpawner` to create user directories. As the hook name implies, this task is accomplished before spawning the end-user's container.
- **Assigning end-user notebook images by role**: this setup uses the `Spawner.pre_spawn_start` within the customized `FirstUseSpawner` assign the user's notebook image. As the hook name implies, this task is accomplished before starting the end-user's container.

## Environment Variables

The services included with this setup rely on environment variables to work properly. Although the ansible script does add sensible defaults for these environment variables you can override them by either setting the ansible veriable when running the playbook or my manually modifying the environment variable files on the remote host after the playbook has run.

### Environment Variables pertaining to JupyterHub, located in `env.jhub`

| Variable  |  Type | Description | Default Value |
|---|---|---|---|
| CONFIGURABLE_HTTP_PROXY |`string` | Random string used to authenticate the proxy with JupyterHub and vs. | `<random_string_value>` |
| DOCKER_LEARNER_IMAGE |`string` | Docker image used by users with the Learner role. | `myedu/notebook:learner` |
| DOCKER_GRADER_IMAGE |`string` | Docker image used by users with the Grader role. | `myedu/notebook:grader` |
| DOCKER_INSTRUCTOR_IMAGE |`string` | Docker image used by users with the Instructor role. | `custom/notebook:instructor` |
| DOCKER_STANDARD_IMAGE |`string` | Docker image used by users with no role. | `myedu/notebook:standard` |
| DOCKER_NETWORK_NAME |`string` | Docker network name for docker-compose and dockerspawner | `jupyter-network` |
| DOCKER_NOTEBOOK_DIR |`string` | Working directory for Jupyter Notebooks | `/home/jovyan` |
| EXCHANGE_DIR |`string` | Exchange directory path  | `myedu.example.com` |
| JUPYTERHUB_CRYPT_KEY |`string` | Cyptographic key used to encrypt cookies. | `<random_value>` |
| JUPYTERHUB_API_TOKEN |`string` | API token used to authenticate grader service with JupyterHub. | `<random_value>` |
| JUPYTERHUB_API_TOKEN_USER |`string` | Grader service user which owns JUPYTERHUB_API_TOKEN. | `grader-{course_id}` |
| JUPYTERHUB_API_URL |`string` | Internal API URL corresponding to JupyterHub. | `http://jupyterhub:8081` |
| ORGANIZATION_NAME |`string` | Organization name. | `test` |
| LTI_CONSUMER_KEY |`string` | LTI 1.1 consumer key | `""` |
| LTI_SHARED_SECRET |`string` | LTI 1.1 shared secret | `""` |

### Environment Variables pertaining to grader service, located in `env.service`

| Variable  |  Type | Description | Default Value |
|---|---|---|---|
| JUPYTERHUB_API_TOKEN |`string` | API token used to authenticate grader service with JupyterHub. | `<random_value>` |
| JUPYTERHUB_BASE_URL |`string` | JupyterHub base URL | `https://<org_name>.example.com/` |
| JUPYTERHUB_SERVICE_URL |`string` | Grader service internal URL | `http://<org_name>:8888` |
| JUPYTERHUB_SERVICE_NAME |`string` | JupyterHub internal service name | `jupyterhub` |
| JUPYTERHUB_API_URL |`string` | JupyterHub API URL | `http://jupyterhub:8081/hub/api` |
| JUPYTERHUB_CLIENT_ID |`string` | JupyterHub Client ID | `service-<course_id>` |
| JUPYTERHUB_SERVICE_PREFIX |`string` | JupyterHub service prefix | `/services/<course_id>` |
| JUPYTERHUB_USER |`string` | JupyterHub user | `grader-<course_id>` |
| NB_USER |`string` | Jupyter Notebook user| `grader-<course_id>` |
| NB_UID=10001 |`string` | Jupyter Notebook Grader UID | `10001` |
| NB_GID=100 |`string` | JupyterHub Notebook Grader GID | `100` |

## Credits

- [JupyterHub for Education](https://jupyterhub-deploy-teaching.readthedocs.io/en/latest/)
- [JupyterHub deployment with docker-compose](https://github.com/jupyterhub/jupyterhub-deploy-docker)
- [nbgrader demo for multiple classes](https://github.com/jupyter/nbgrader/tree/master/demos/demo_multiple_classes)
