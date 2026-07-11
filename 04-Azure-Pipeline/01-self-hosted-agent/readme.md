# Self Hosted Agent

https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/linux-agent?view=azure-devops&tabs=IP-V4

- In the self hoste agent pool we can select specific agent to run the pipeline. This is useful when we want to run a pipeline on a specific machine with specific hardware or software requirements. For that we can use `capabilities` (user defined and system capabilities).

```yaml
pool:
  name: "treinetic-local-agent-pool"
  # select the agent with the name "dev-agent" from the pool
  demands:
    - Agent.Name -equals dev-agent
```

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

```bash
sudo useradd \
    --system \
    --create-home \
    --home-dir /home/azureagent \
    --shell /usr/sbin/nologin \
    azureagent
```

```bash
sudo mkdir -p /opt/azureagent
```

Download the agent and extract it to /opt/azureagent

```bash
sudo chown -R azureagent:azureagent /opt/azureagent
```

```bash
sudo su -s /bin/bash azureagent
```

temporarily gives the azureagent user a Bash shell for that session only.

```
Switch to azureagent
↓
Ignore /usr/sbin/nologin
↓
Start /bin/bash
↓
You get a shell as azureagent
```

```bash
./config.sh
```

### use microk8s from azureagent

- MicroK8s is installed as a Snap package. Snap applications expect every user to have a writable home directory because they store per-user data under:

```
/home/<username>/snap/
```

- So we must create a writable home directory for the azureagent user. The home directory is created in the useradd command above.

- also to use microk8s kubectl without sudo, we need to add the azureagent user to the microk8s group:

```bash
sudo usermod -a -G microk8s azureagent
newgrp microk8s
```
