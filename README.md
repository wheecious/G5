# Centralized Logs and Metrics Collection Service

This repository provides instructions for setting up a Grafana-based monitoring service to collect, store, and display logs and metrics. The setup can be achieved either through a fully automated GitLab CI pipeline or by manually running an Ansible playbook.

## Overview

The service uses Grafana, Prometheus, and Filebeat to create a centralized system for monitoring logs and metrics. This README outlines two methods for installation: using a GitLab runner for automation or manually executing the Ansible playbook.

## Prerequisites

- A Debian-based host (e.g., Ubuntu) to deploy the services.
- SSH access to the target host with a user possessing sufficient privileges (see **User Privileges** below).
- Ansible installed (for manual setup) or a GitLab runner configured (for automated setup).

## Installation Methods

### Option 1: Using a GitLab Runner (Automated)

The repository includes a `.gitlab-ci.yml` file that defines a CI pipeline to automate the setup using the `python:latest` Docker image. The pipeline performs the following tasks:

1. Configures the SSH agent.
2. Installs the Ansible package.
3. Generates a `.env` file with Grafana credentials.
4. Creates a file in the `host_vars` directory with the user for running the playbook.
5. Configures the `root_url` in the `grafana.ini` file.
6. Runs the Ansible playbook to set up the services.

#### Required GitLab CI Variables

| Variable              | Description                                                                 |
|-----------------------|-----------------------------------------------------------------------------|
| `SSH_PRIVATE_KEY`     | SSH private key in base64 format (decoded in the pipeline).                 |
| `PROJECT_USER`        | User with SSH access to the host and permissions to run the playbook.       |
| `PROJECT_HOST`        | The target host where services will be deployed.                            |
| `GRAFANA_USER`        | Grafana admin username (should be `admin`).                                |
| `GRAFANA_PASS`        | A secure password for the Grafana admin user.                              |
| `GRAFANA_INI_ROOT_URL`| The `root_url` for Grafana (e.g., `http://your-grafana-host`).             |

#### User Privileges
The `PROJECT_USER` must have:
- **SSH access** to the target host (`PROJECT_HOST`).
- **Passwordless `sudo` access** (not recommended for production) or explicit permissions for:
  - Installing packages (`apt`).
  - Managing system services (`docker`, `nginx`).
  - Modifying system files (`/etc/nginx/`, `/opt/monitoring/`).
  - Adding users to the `docker` group.

#### Steps
1. Store the required variables in GitLab CI/CD settings (ensure `SSH_PRIVATE_KEY` is masked for security).
2. Push the repository to GitLab and let the pipeline run.
3. Verify the services are running on the target host.

### Option 2: Manual Setup with Ansible Playbook

If you prefer manual setup or lack a GitLab runner, you can run the Ansible playbook directly.

#### Steps
1. **Prepare the Inventory File**:
   - Create an `inventory` file in the same directory as the playbook.
   - Add the target host and user with necessary privileges:
     ```
     grafana ansible_host=<your_grafana_host> ansible_user=<user_with_necessary_privileges>
     ```

2. **Prepare Configuration Files**:
   - Create a `.env` file in the `files/` directory with the following content:
     ```
     GF_SECURITY_ADMIN_USER=admin
     GF_SECURITY_ADMIN_PASSWORD=<your_grafana_password>
     ```
   - Create a `grafana.ini` file in the `files/` directory with:
     ```
     [server]
     serve_from_sub_path = true
     root_url = http://<your_grafana_host_name>/
     ```

3. **Run the Playbook**:
   - Execute the following command from the directory containing the playbook:
     ```
     ansible-playbook playbook.yml -i inventory -v
     ```

4. **User Privileges**:
   - The `ansible_user` must have:
     - SSH access to the target host.
     - Passwordless `sudo` or explicit permissions for package installation, file management, service management, and Docker group membership.

## Adding Prometheus Data Source in Grafana

After the services are set up, configure Grafana to use Prometheus as a data source:

1. Open a browser and navigate to your Grafana address (e.g., `http://<your_grafana_host_name>`).
2. Log in with the credentials: `admin/<your_grafana_password>`.
3. Go to **Grafana Menu > Connections > Data sources**.
4. Click **Add new data source** and select **Prometheus**.
5. Configure the data source:
   - Set the Prometheus URL (e.g., `http://<prometheus_host>:9090`).
   - Check **Skip TLS certificate validation**.
   - Set **Timeout** to `15` seconds.
6. Click **Save and test** to verify the connection.
7. If successful, click **Build a dashboard** (top right).
8. Select **Add visualization** > **Prometheus**.
9. Use the `json_filebeat` job for Filebeat metrics: `{job="json_filebeat"}`.
10. Click **Run query** and **Save dashboard**.

## Notes
- **Security**: Avoid using passwordless `sudo` in production. Instead, configure explicit `sudo` permissions for required operations.
- **Grafana Password**: Use a strong, secure password for `GRAFANA_PASS`.
- **SSH Key**: Store the `SSH_PRIVATE_KEY` securely in GitLab CI variables or locally for manual setup.
- **Metrics**: The playbook configures Filebeat metrics under the `json_filebeat` job in Prometheus.

## Troubleshooting
- **Pipeline Failure**: Check GitLab CI logs for missing variables or SSH issues.
- **Playbook Errors**: Ensure the `ansible_user` has sufficient privileges and the `inventory` file is correct.
- **Grafana Access**: Verify the `root_url` in `grafana.ini` matches the host address.
- **Prometheus Connection**: Confirm the Prometheus service is running and accessible.

For further assistance, refer to the official documentation for [Grafana](https://grafana.com/docs/), [Prometheus](https://prometheus.io/docs/), and [Ansible](https://docs.ansible.com/).