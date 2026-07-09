# Self Hosted Agent

https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4

## Prepare permissions

- The user configuring the agent needs pool admin permissions
- The folders controlled by the agent should be restricted to as few users as possible because they contain secrets that could be decrypted or exfiltrated. ( because it contains the PAT token and other secrets)
- It makes sense to grant access to the agent folder only for DevOps administrators and the user identity running the agent process.
- If you run your agent as a service, you cannot run the agent service as root user

```bash
sudo ./svc.sh install [username]
```

This command creates a service file that points to ./runsvc.sh. This script sets up the environment (more details below) and starts the agents host. If username parameter is not specified, the username is taken from the $SUDO_USER environment variable set by sudo command. This variable is always equal to the name of the user who invoked the sudo command.

```bash
sudo ./svc.sh start
```

```bash
sudo ./svc.sh uninstall
```

# ---------------------------------

```
sudo useradd \
  --system \
  --create-home \
  --home-dir /opt/azureagent \
  --shell /usr/sbin/nologin \
  azureagent
```

- Download and extract the agent to a folder on your machine.
