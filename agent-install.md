# Using Packer to create a custom AMI for Persistent Runners (without auditd)

---

Use this Packer template **only** when you operate long-lived self-hosted GitHub Actions runners that are reused across jobs and do **not** use auditd-based monitoring.

AWS EC2 users can create a custom AMI using the Packer template provided below. If you've previously created a custom AMI for GitHub Actions runner VMs using a different packer template, modify that template according to the steps outlined below. Ensure packer.json and setup.sh are located in the same directory. For documentation, we have placed `packer.json` and `setup.sh` inside `packer` directory.

### packer.json

```json
{
  "variables": {
    "region": "",
    "source_ami": "",
    "vpc_id": "",
    "subnet_id": "",
    "security_group": "",
    "api_key": ""
  },
  "sensitive-variables": ["api_key"],
  "builders": [
    {
      "type": "amazon-ebs",
      "region": "{{user `region`}}",
      "source_ami": "{{user `source_ami`}}",
      "instance_type": "t3.large",
      "ssh_username": "ubuntu",
      "ami_name": "github-actions-runner-{{isotime \"2006-01-02T15-04-05Z07:00\"}}",
      "ssh_timeout": "5m",
      "associate_public_ip_address": true,
      "vpc_id": "{{user `vpc_id`}}",
      "subnet_id": "{{user `subnet_id`}}",
      "security_group_ids": ["{{user `security_group`}}"]
    }
  ],
  "provisioners": [
    {
      "type": "shell",
      "script": "./packer/setup.sh",
      "environment_vars": ["API_KEY={{user `api_key`}}"]
    }
  ]
}
```

### setup.sh

Here is the template you can use for setting up Harden-Runner in your custom AWS AMI. Append this snippet to your existing AMI provisioning script if you have one.

```bash
# 1. Create "/home/agent" directory if it does not exist.
sudo mkdir -p /home/agent
sudo chown -R $USER /home/agent
cd /home/agent

# 2. Download agent binary and store it at /home/agent
architecture=$(uname -m)
if [ "$architecture" = "aarch64" ]; then
    agent_tar="harden-runner-bravo_1.7.15_linux_arm64.tar.gz"
else
    agent_tar="harden-runner-bravo_1.7.15_linux_amd64.tar.gz"
fi

curl -O "https://packages.stepsecurity.io/self-hosted/$agent_tar"

# 3. Confirm checksum
if [ "$architecture" = "aarch64" ]; then
    echo "410ab117b7d1919b983e639129bfd2793ee1a23b93a490efacd4673cc8e5ae57  $agent_tar" | sha256sum -c
else
    echo "4d8834a405b592f29d578e7b3085f9e317d318b00ee1901c4b245465853ce319  $agent_tar" | sha256sum -c
fi

# 4. Extract tar
tar -xzf "$agent_tar"

# 5. Create /home/agent/agent.json with the provided content.
cat <<EOL > /home/agent/agent.json
{
  "customer": "{{OWNER}}",
  "runner_work_directory": "{{GITHUB_ACTIONS_WORKING_DIRECTORY}}",
  "api_key": "$API_KEY",
  "api_url": "https://agent.api.stepsecurity.io/v1"
}
EOL

# 6. Make agent executable.
chmod +x /home/agent/agent

# 7. Copy agent.service
sudo cp agent.service /etc/systemd/system/agent.service

# 8. Enable and start the service so that it automatically starts on startup.
sudo systemctl daemon-reload
sudo systemctl enable agent
```

> **Note**: The script requires the working directory for the GitHub Actions runner binary. Replace `{{GITHUB_ACTIONS_WORKING_DIRECTORY}}` with the exact path where you've installed the GitHub Actions runner binary. You can define a variable for the working directory during the GitHub Actions runner binary installation in your custom AMI and then reference it as shown in the example below.

**Example:**

```bash
export ACTIONSRUNNERDIR="/home/actions/actions-runner"
# setup runner
sudo mkdir -p $ACTIONSRUNNERDIR
sudo chown -R $USER $ACTIONSRUNNERDIR
cd $ACTIONSRUNNERDIR
curl -o actions-runner-linux-x64-2.309.0.tar.gz -L https://github.com/actions/runner/releases/download/v2.309.0/actions-runner-linux-x64-2.309.0.tar.gz
...
# 5. Create /home/agent/agent.json with the provided content.
cat <<EOL > /home/agent/agent.json
{
  "customer": "step-security",
  "runner_work_directory": "$ACTIONSRUNNERDIR",
  "api_key": "$API_KEY",
  "api_url": "https://agent.api.stepsecurity.io/v1"
}
EOL
```

### Creating a Custom AMI with Packer

Run the following command to use Packer to create a custom Amazon Machine Image (AMI).

```bash
packer build \
  -var 'region={{AWS_REGION}}' \
  -var 'source_ami={{BASE_AMI}}' \
  -var 'vpc_id={{VPC_ID}}' \
  -var 'subnet_id={{SUBNET_ID}}' \
  -var 'security_group={{SECURITY_GROUP_ID}}' \
  -var 'api_key=step_9d6f4942-57e9-400d-9182-a6c40a90b913' \
  packer.json
```

### Parameters

When creating a custom AMI, Packer will temporarily spin up a new AWS EC2 instance. You need to provide the following configuration parameters:

- **Security Group Configuration**: Ensure the AWS security group you use permits SSH access to the EC2 instance from the machine where the Packer command is executed. Replace the `{{VPC_ID}}`, `{{SUBNET_ID}}`, and `{{SECURITY_GROUP_ID}}` placeholders as per your chosen security group.
- **Base AMI Selection**: Decide on a base AMI that aligns with your engineering policy. Replace `{{BASE_AMI}}` with the AMI ID of your choice.
