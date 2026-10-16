# CodeAlpha Jenkins Remoting Project

Distributed Jenkins setup for the CodeAlpha DevOps Internship (Task 2). A Jenkins controller runs on one AWS EC2 instance and builds are executed remotely on a separate EC2 agent connected over SSH.

## Architecture

| Component | Details |
|---|---|
| Controller | EC2 t3.small, Ubuntu, Jenkins LTS, Java 21 |
| Agent | EC2 t3.micro, Ubuntu, Java 21, Docker, Git |
| Connection | Controller to agent over SSH (port 22, private IP, same VPC) |
| Agent label | `docker-agent` |
| Pipeline | `Jenkinsfile` in this repo |

Controller access is through an SSH tunnel, so the Jenkins UI port is not open to the internet.

## What this project covers

- Remote Jenkins nodes connected with the SSH Build Agents plugin
- Build load moved off the controller to a remote machine
- Jobs run remotely on the agent (confirmed by hostname in the build log)
- Node isolation with a label and the setting "Only build jobs with label expressions matching this node"
- Secure connection using an SSH key pair stored in Jenkins credentials

## Setup steps

1. Install Java and Jenkins on the controller.

        sudo apt update && sudo apt install -y fontconfig openjdk-21-jre-headless
        sudo mkdir -p /etc/apt/keyrings
        sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
        echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
        sudo apt update && sudo apt install -y jenkins
        sudo systemctl enable --now jenkins

2. Access the Jenkins UI through an SSH tunnel, then open http://localhost:8081

        ssh -i <key>.pem -N -L 8081:localhost:8080 ubuntu@<CONTROLLER-PUBLIC-IP>

3. Prepare the agent machine.

        sudo apt update && sudo apt install -y openjdk-21-jre-headless git docker.io
        sudo systemctl enable --now docker
        sudo usermod -aG docker ubuntu
        mkdir -p /home/ubuntu/jenkins-agent

4. Create an SSH key pair on the controller and add the public key to the agent.

        ssh-keygen -t ed25519 -f ~/.ssh/jenkins_agent -N ""
        cat ~/.ssh/jenkins_agent.pub
        # on the agent, append the public key to ~/.ssh/authorized_keys

5. In Jenkins, add the private key as a credential (Kind: SSH Username with private key, ID: `agent-ssh-key`, user: `ubuntu`). The private key is never stored in this repository.

6. Add the node in Jenkins (Manage Jenkins, Nodes, New Node):

   - Type: Permanent Agent
   - Remote root directory: `/home/ubuntu/jenkins-agent`
   - Label: `docker-agent`
   - Usage: Only build jobs with label expressions matching this node
   - Launch method: Launch agents via SSH (agent private IP, credential `agent-ssh-key`)

7. Create a Pipeline job that uses the `Jenkinsfile` from this repo.

## Pipeline

The pipeline runs only on the `docker-agent` node and has four stages:

1. Node info (hostname, architecture, user) to prove remote execution
2. Clone the Docker web server repo
3. Docker build
4. Run the container and test it with curl, then clean up

## Result

The build log shows `Running on docker-agent in /home/ubuntu/jenkins-agent/workspace/...`, the hostname of the agent EC2, and ends with `Finished: SUCCESS`.

## Security notes

- SSH key authentication only, no passwords
- Jenkins UI reached through an SSH tunnel, not exposed publicly
- Agent security group allows SSH only from the controller security group
- Node isolation with labels, so unrelated jobs do not run on the agent
- No secrets or keys are committed to this repository
- Host key verification is set to "Non verifying" for this private VPC lab. For production, use "Manually trusted key" verification.

## Issues faced and fixes

| Issue | Fix |
|---|---|
| `apt update` failed with `NO_PUBKEY` for the Jenkins repo | Jenkins changed its repo signing key, so I switched to `jenkins.io-2026.key` |
| Agent log showed "Key exchange was not finished, connection is closed" | Host key verification was set to Known hosts file with no known_hosts file. Changed the strategy for this lab setup |
| Agent went offline with a disk space warning | The agent `/tmp` was only about 454 MiB, below the default temp space threshold. Lowered the temp space thresholds in the node settings |
| Jenkins UI not reachable from the browser | Used an SSH tunnel instead of opening port 8080 |

## Future improvements

- Add an ARM agent to run jobs on a different architecture
- Use "Manually trusted key" host key verification
- Add parallel stages to spread work across multiple agents

## Screenshots

Screenshots are in the `screenshots` folder.
