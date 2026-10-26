# AIML DevOps PBL — All 5 Sets

Step-by-step solutions for all 15 questions in the assignment. GitHub displays a **Copy** button on code blocks, so you can copy commands directly into your EC2 terminal.

> **Before running anything:** replace `YOUR_REPO_URL`, `YOUR_REPO_NAME`, `YOUR_GITHUB_USERNAME`, `YOUR_EMAIL`, `EC2_PUBLIC_IP`, and key paths with your own values. These steps assume Ubuntu on AWS EC2. Tool versions and university lab environments may differ.

## Contents
- [Prerequisites](#prerequisites)
- [Set 1](#set-1)
- [Set 2](#set-2)
- [Set 3](#set-3)
- [Set 4](#set-4)
- [Set 5](#set-5)
- [Verification and troubleshooting](#verification-and-troubleshooting)

## Prerequisites

### Connect to EC2 from your computer

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

For Amazon Linux, the login user is often `ec2-user`. Restrict SSH (port 22) to your own IP. Only open other ports when needed for the practical.

### Install common tools and configure Git

```bash
sudo apt update
sudo apt install -y git curl unzip ca-certificates openjdk-17-jdk maven
git config --global user.name "YOUR_NAME"
git config --global user.email "YOUR_EMAIL"
git config --global init.defaultBranch main
java -version
mvn -version
git --version
```

**GitHub security:** do not put passwords, access tokens, AWS credentials, or `.pem` private keys in this README or your repository. Use GitHub's supported HTTPS authentication or configure SSH keys securely.

### Sample Java project used in the examples

```bash
mkdir -p ~/projects/HelloWorld
cd ~/projects/HelloWorld
cat > HelloWorld.java <<'EOF'
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello from EC2!");
    }
}
EOF
javac HelloWorld.java
java HelloWorld
```

Expected output: `Hello from EC2!`

---

# Set 1

## Q1 — Linux & Git: Create project, set permissions, commit and push

```bash
mkdir -p ~/projects/HelloWorld
cd ~/projects/HelloWorld
cat > HelloWorld.java <<'EOF'
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello from EC2!");
    }
}
EOF

chmod 750 ~/projects/HelloWorld
ls -ld ~/projects/HelloWorld

git init
git add HelloWorld.java
git commit -m "Add HelloWorld Java program"
git branch -M main
git remote add origin YOUR_REPO_URL
git push -u origin main
```

If `origin` already exists, use `git remote set-url origin YOUR_REPO_URL` instead of `git remote add`. Verify with:

```bash
git status
git log -1 --oneline
git remote -v
ls -ld ~/projects/HelloWorld
```

`750` gives the owner read/write/execute, the group read/execute, and others no permissions. Expected: the file and commit appear in GitHub.

## Q2 — Maven: Create a project and generate a JAR

```bash
mkdir -p ~/projects/hello-maven/src/main/java/com/example
cd ~/projects/hello-maven
cat > pom.xml <<'EOF'
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>hello-maven</artifactId>
  <version>1.0.0</version>
  <properties>
    <maven.compiler.release>17</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
</project>
EOF

cat > src/main/java/com/example/App.java <<'EOF'
package com.example;
public class App {
    public static void main(String[] args) {
        System.out.println("Maven build successful!");
    }
}
EOF

mvn clean package
ls -lh target/
```

Expected: `BUILD SUCCESS` and `target/hello-maven-1.0.0.jar`. This minimal project demonstrates packaging; it does not configure a runnable JAR manifest.

Optional Ansible package install ad-hoc command, run from a control host with SSH access:

```bash
ansible all -i "EC2_PUBLIC_IP," -u ubuntu --private-key ./your-key.pem \
  -b -m apt -a "name=maven state=present update_cache=yes"
```

## Q3 — Docker & Kubernetes: Build image and create a Pod

Install Docker:

```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"
```

Log out and reconnect for group membership to apply; alternatively use `sudo docker`.

Create a simple Dockerfile in the Maven project:

```bash
cd ~/projects/hello-maven
cat > Dockerfile <<'EOF'
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY src/main/java/com/example/App.java /app/App.java
RUN javac App.java
CMD ["java", "App"]
EOF

docker build -t hello-maven:1.0 .
docker images
docker run --rm hello-maven:1.0
```

For Kubernetes, install `kubectl` and Minikube using their official guides, then start Minikube with a supported driver. On a compatible Docker setup:

```bash
minikube start --driver=docker
minikube image load hello-maven:1.0
kubectl run hello-pod --image=hello-maven:1.0 --image-pull-policy=IfNotPresent
kubectl get pods
kubectl describe pod hello-pod
kubectl logs hello-pod
```

The Java program prints once and exits, so its Pod can show `Completed` rather than `Running`. If the practical strictly needs a Running Pod, use a long-running server image:

```bash
kubectl run nginx-pod --image=nginx:stable --port=80
kubectl get pods -w
```

---

# Set 2

## Q1 — Jenkins Freestyle job: Pull GitHub code and build with Maven

1. Install Jenkins by following the current official Jenkins Ubuntu instructions. Ensure Java is installed.
2. Start and verify Jenkins:

```bash
sudo systemctl enable --now jenkins
sudo systemctl status jenkins --no-pager
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

3. Open `http://EC2_PUBLIC_IP:8080` only after restricting inbound port 8080 to your IP. Complete the setup wizard and install suggested plugins.
4. Push the Maven project from Set 1 Q2 to GitHub; `pom.xml` must be in the repository root.
5. In Jenkins click **New Item → Freestyle project → OK**.
6. Under **Source Code Management**, choose **Git**, enter `YOUR_REPO_URL`, add credentials if the repo is private, and set the branch to `*/main`.
7. Under **Build Steps**, select **Invoke top-level Maven targets** if available and enter `clean package`. Alternatively use **Execute shell**:

```bash
mvn -version
mvn clean package
test -f target/hello-maven-1.0.0.jar
```

8. Click **Save → Build Now → Console Output**.

Expected: build status `SUCCESS` and Maven reports `BUILD SUCCESS`. Do not expose an unsecured Jenkins instance publicly.

## Q2 — GitHub webhook and Ansible service check

### Configure the webhook
1. Install/configure the GitHub plugin in Jenkins.
2. Open the job's **Configure → Build Triggers** and enable **GitHub hook trigger for GITScm polling**.
3. In GitHub go to **Repository Settings → Webhooks → Add webhook**.
4. Set payload URL to `http://EC2_PUBLIC_IP:8080/github-webhook/` only for a deliberately accessible lab endpoint. Prefer a secured HTTPS setup; never leave Jenkins with default credentials or broad unnecessary access.
5. Choose `application/json` and **Just the push event**, then add the webhook.
6. Push a commit. Check GitHub webhook delivery status and Jenkins build history.

### Install Ansible and create `services.yml`

```bash
sudo apt update
sudo apt install -y ansible
cat > services.yml <<'EOF'
---
- name: Verify Jenkins and Docker
  hosts: localhost
  connection: local
  become: true
  tasks:
    - name: Check Jenkins active
      ansible.builtin.command: systemctl is-active jenkins
      changed_when: false
    - name: Check Docker active
      ansible.builtin.command: systemctl is-active docker
      changed_when: false
    - name: Enable Jenkins at boot
      ansible.builtin.service:
        name: jenkins
        enabled: true
    - name: Enable Docker at boot
      ansible.builtin.service:
        name: docker
        enabled: true
EOF

ansible-playbook services.yml
```

Run this on a host where both services are installed. Expected: services report `active`, and Ansible completes with zero failures.

## Q3 — Docker & Linux: Run detached container and inspect logs/processes

```bash
docker run -d --name web-demo -p 8081:80 nginx:stable
docker ps
docker logs web-demo
docker inspect web-demo
ps aux | grep '[n]ginx'
top
```

If port 8081 is allowed from your IP in the EC2 security group, test `http://EC2_PUBLIC_IP:8081`. Clean up:

```bash
docker stop web-demo
docker rm web-demo
```

Expected: container is visible in `docker ps`; logs and Linux process information are available.

---

# Set 3

## Q1 — Git & Linux: Clone repository, create branch, modify and push

```bash
mkdir -p ~/projects
cd ~/projects
git clone YOUR_REPO_URL
cd YOUR_REPO_NAME
git switch -c feature/update-hello
```

Edit the Java file (adjust the path if it is not in the repository root):

```bash
cat > HelloWorld.java <<'EOF'
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Feature branch update from EC2!");
    }
}
EOF

git add HelloWorld.java
git commit -m "Update HelloWorld output"
git push -u origin feature/update-hello
git rev-parse --short HEAD
git status
```

Expected: feature branch, updated code, and commit hash appear on GitHub.

## Q2 — Jenkins Declarative Pipeline: Checkout → Build → Test

Add a `Jenkinsfile` to the Maven repository:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B clean package'
            }
        }
        stage('Test') {
            steps {
                sh 'test -f target/hello-maven-1.0.0.jar'
                sh 'echo "Artifact check passed"'
            }
        }
    }
    post {
        success { echo 'Pipeline completed successfully.' }
        failure { echo 'Pipeline failed. Check the logs.' }
    }
}
```

The Test stage checks that the JAR exists; it is not a unit-test suite. If actual tests exist, run `mvn test`.

```bash
git add Jenkinsfile
git commit -m "Add declarative Jenkins pipeline"
git push
```

In Jenkins: **New Item → Pipeline → OK → Pipeline script from SCM → Git**. Enter repo URL, credentials if needed, branch `*/main`, and Script Path `Jenkinsfile`. Save, build, then inspect **Stage View** and **Console Output**.

Expected: Checkout, Build, and Test stages succeed.

## Q3 — Docker tag `app:v1` and generate `pod.yaml`

Run from the directory containing the Dockerfile:

```bash
docker build -t app:v1 .
docker images app
docker run --rm app:v1
kubectl run app-pod --image=app:v1 --image-pull-policy=IfNotPresent \
  --dry-run=client -o yaml > pod.yaml
cat pod.yaml
```

For Minikube, load the local image and apply the generated manifest:

```bash
minikube image load app:v1
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod app-pod
kubectl logs app-pod
```

Expected: `app:v1` appears in Docker images and a valid `pod.yaml` is saved. If a Pod with that name already exists, use a different name or inspect/remove the old resource first.

---

# Set 4

## Q1 — Linux & Git: Create `devopsuser` and push a project

```bash
sudo adduser devopsuser
getent passwd devopsuser
grep '^devopsuser:' /etc/passwd
sudo -iu devopsuser
mkdir -p ~/devops-app
cd ~/devops-app
git config --global user.name "YOUR_NAME"
git config --global user.email "YOUR_EMAIL"
```

Create a Java file and push:

```bash
cat > HelloWorld.java <<'EOF'
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello from devopsuser!");
    }
}
EOF

git init
git add HelloWorld.java
git commit -m "Initial commit from devopsuser"
git branch -M main
git remote add origin YOUR_REPO_URL
git push -u origin main
exit
```

If the remote already has commits or `origin` already exists, inspect `git remote -v` and the log before changing anything. Expected: user is in `/etc/passwd`, and the initial commit is on GitHub.

## Q2 — Ansible & Jenkins: Install Nginx, deploy status page, run from Jenkins

Install Ansible on the control host:

```bash
sudo apt update
sudo apt install -y ansible
mkdir -p ~/ansible-lab
cd ~/ansible-lab
```

Create a localhost inventory for a same-machine lab:

```bash
cat > inventory.ini <<'EOF'
[web]
localhost ansible_connection=local
EOF
```

For a separate target host, use an inventory like this instead and secure the private key:

```ini
[web]
web1 ansible_host=EC2_PUBLIC_IP ansible_user=ubuntu ansible_ssh_private_key_file=/secure/path/your-key.pem
```

Create `setup.yml`:

```bash
cat > setup.yml <<'EOF'
---
- name: Install Nginx and deploy status page
  hosts: web
  become: true
  tasks:
    - name: Install Nginx
      ansible.builtin.apt:
        name: nginx
        state: present
        update_cache: true
    - name: Deploy status page
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: |
          <!doctype html>
          <html><head><title>DevOps PBL</title></head>
          <body><h1>Deployment successful!</h1><p>Managed by Ansible.</p></body></html>
        owner: root
        group: root
        mode: "0644"
    - name: Enable and start Nginx
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
EOF

ansible-playbook -i inventory.ini setup.yml
curl -I http://localhost
curl http://localhost
```

Run the playbook a second time to demonstrate idempotence; if nothing changed, Ansible should report few or no changes.

### Run it from Jenkins
1. Commit the playbook and inventory to your lab repository (never commit a private key).
2. Create a Jenkins **Freestyle project** and configure Git checkout.
3. Add an **Execute shell** build step:

```bash
ansible-playbook -i inventory.ini setup.yml
```

4. Save, build, and inspect **Console Output**.

Jenkins may run as the `jenkins` user, which may not have permission to use `sudo`. Configure access securely rather than granting broad passwordless root access. Expected: Ansible has zero failures and the web page is served on the target.

## Q3 — Kubernetes: Deployment with two replicas

For a simple long-running demonstration, use Nginx:

```bash
minikube start --driver=docker
kubectl create deployment nginx-deployment --image=nginx:stable
kubectl scale deployment nginx-deployment --replicas=2
kubectl get deployments
kubectl get pods -o wide
kubectl describe deployment nginx-deployment
```

If using the locally built `app:v1`, first run `minikube image load app:v1`, then create and scale the Deployment using that image. A Java program that prints once exits, so its Pods may not remain Running; use a server image to demonstrate two Running replicas.

Expected: Deployment shows `2/2` available and two Pods are listed.

---

# Set 5

## Q1 — Git, Linux & Docker: Clone, build multi-command Dockerfile and test

```bash
mkdir -p ~/projects
cd ~/projects
git clone YOUR_REPO_URL
cd YOUR_REPO_NAME
```

For a `HelloWorld.java` file at the repository root:

```bash
cat > Dockerfile <<'EOF'
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY HelloWorld.java .
RUN javac HelloWorld.java
CMD ["java", "HelloWorld"]
EOF

docker build -t pbl-java-app:1.0 .
docker images pbl-java-app
docker run --name pbl-java-test pbl-java-app:1.0
docker ps -a
docker logs pbl-java-test
docker rm pbl-java-test
```

If the source file is in a subdirectory, update the `COPY` path. Since this program prints and exits, use `docker ps -a` to see the exited container; `docker ps` lists only running containers. Expected: image builds and application output is visible in logs.

## Q2 — Jenkins Pipeline: Build project/image and run Ansible health check

Add a `Jenkinsfile` to a repository containing a compatible `pom.xml` and `Dockerfile`:

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Maven Build') {
            steps { sh 'mvn -B clean package' }
        }
        stage('Docker Build') {
            steps { sh 'docker build -t pbl-java-app:ci .' }
        }
        stage('Ansible Health Check') {
            steps {
                sh '''
                  ansible localhost -i "localhost," -c local \
                    -m ansible.builtin.ping
                '''
            }
        }
    }
    post {
        success { echo 'Build, Docker image, and Ansible check succeeded.' }
        failure { echo 'A stage failed. Review the console log.' }
    }
}
```

The Jenkins agent needs Git, Java, Maven, Docker, and Ansible configured; the Jenkins execution account must have permission to run Docker.

```bash
git add Jenkinsfile
git commit -m "Add CI pipeline with Docker and Ansible check"
git push
```

In Jenkins create a **Pipeline** job and choose **Pipeline script from SCM**. Enter the Git repository, branch `*/main`, and Script Path `Jenkinsfile`. Build and inspect the console. The localhost ping checks local Ansible execution; for a remote node, configure an inventory and SSH credentials and ping the target group.

Expected: checkout, Maven build, Docker build, and Ansible health check succeed.

## Q3 — Kubernetes: Deploy and expose a NodePort Service on service port 8080

Use Nginx as a long-running HTTP server:

```bash
kubectl create deployment web-deployment --image=nginx:stable
kubectl rollout status deployment/web-deployment
kubectl get deployments
kubectl get pods
```

Expose it using Service port 8080 and target container port 80:

```bash
kubectl expose deployment web-deployment \
  --type=NodePort \
  --port=8080 \
  --target-port=80 \
  --name=web-service
```

Verify and get a Minikube URL:

```bash
kubectl get svc web-service
kubectl describe svc web-service
kubectl get endpoints web-service
minikube service web-service --url
```

Open the URL returned by Minikube. Depending on the driver, the command may need to remain running to maintain a tunnel.

**Port clarification:** Kubernetes commonly allocates a NodePort in the default range `30000–32767`. The Service port in this example is `8080`, while the actual NodePort is shown after the colon in `kubectl get svc`. If the assessment literally requires NodePort number 8080, the cluster's NodePort range must be changed, which is an advanced cluster configuration.

---

# Verification and troubleshooting

## Useful commands

### Linux and Git

```bash
whoami
ls -ld ~/projects/HelloWorld
git status
git log --oneline -5
git branch
git remote -v
```

### Java and Maven

```bash
java -version
mvn -version
mvn clean package
find target -maxdepth 1 -type f -name '*.jar' -ls
```

### Jenkins

```bash
sudo systemctl status jenkins --no-pager
sudo journalctl -u jenkins -n 50 --no-pager
```

### Ansible

```bash
ansible --version
ansible-playbook -i inventory.ini setup.yml --syntax-check
ansible-playbook -i inventory.ini setup.yml
```

### Docker

```bash
docker images
docker ps
docker ps -a
docker logs CONTAINER_NAME
```

### Kubernetes

```bash
kubectl get pods -o wide
kubectl get deployments
kubectl get svc
kubectl describe pod POD_NAME
kubectl logs POD_NAME
```

## Common errors

| Error | Likely cause | Fix |
|---|---|---|
| Docker permission denied | User cannot access Docker socket | Use `sudo docker ...` or reconnect after group membership change. Docker group membership is highly privileged. |
| SSH `Permission denied (publickey)` | Wrong key, username, or security group | Verify key path, `chmod 400`, EC2 username, and port 22 rule. |
| `remote origin already exists` | Git remote already configured | Run `git remote -v`, then `git remote set-url origin YOUR_REPO_URL`. |
| Git push authentication fails | GitHub auth not configured | Configure SSH or supported HTTPS credentials. Never commit tokens. |
| `mvn: command not found` | Maven missing | Install Maven and check `mvn -version`. |
| Maven cannot find `pom.xml` | Wrong working directory | `cd` into the directory containing `pom.xml`. |
| Jenkins cannot find Maven/Java | Tools missing from agent or Jenkins configuration | Configure tools and check versions in the job console. |
| GitHub webhook does not trigger | Trigger/plugin/URL/network misconfigured | Check job trigger, webhook delivery, endpoint URL, and network access. |
| Kubernetes `ImagePullBackOff` for local image | Minikube cannot see the host image | Run `minikube image load IMAGE:TAG`; use `IfNotPresent`. |
| Java Pod exits | Program finishes after printing | Expected for a one-shot program. Use Nginx or another long-running service for a Running Pod. |
| Service has no endpoints | Pods not ready or selector mismatch | Check `kubectl get pods --show-labels`, `kubectl describe svc SERVICE`, and readiness. |
| Web page inaccessible | Wrong port, service, or firewall | Check `kubectl get svc`, `minikube service NAME --url`, and EC2 security-group rules. |

## Final submission checklist

- [ ] All five sets and all 15 questions attempted.
- [ ] Git commits, branches, and pushes visible on GitHub.
- [ ] Linux permissions and `devopsuser` verified.
- [ ] Maven shows `BUILD SUCCESS` and JAR exists in `target/`.
- [ ] Jenkins Freestyle and Pipeline jobs show successful Console Output.
- [ ] GitHub webhook automatically triggers a build.
- [ ] Ansible playbook completes with zero failures.
- [ ] Docker image and container state/logs verified.
- [ ] Kubernetes Pods, two-replica Deployment, and Service verified.
- [ ] Screenshots captured if required by the instructor.
- [ ] No secrets, tokens, private keys, or AWS credentials committed.

## Official documentation

- [Git](https://git-scm.com/doc)
- [Apache Maven](https://maven.apache.org/guides/)
- [Jenkins](https://www.jenkins.io/doc/)
- [Ansible](https://docs.ansible.com/)
- [Docker](https://docs.docker.com/)
- [Kubernetes](https://kubernetes.io/docs/)
- [Minikube](https://minikube.sigs.k8s.io/docs/)
- [GitHub](https://docs.github.com/)

**Note:** This guide follows the five-set assignment structure. AWS networking, operating-system images, installed tool versions, Jenkins plugins, and Kubernetes drivers vary by environment. Verify each command's output on your system before claiming a practical is complete.
