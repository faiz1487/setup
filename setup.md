## Java

Install OpenJDK (choose 17 or 21 depending on your needs):

```bash
# OpenJDK 17
sudo apt install -y openjdk-17-jdk

# OR OpenJDK 21
sudo apt install -y openjdk-21-jdk
```
Verify:
```bash
java --version
```

---

## Jenkins

Official docs: https://www.jenkins.io/doc/book/installing/linux/

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```
Initial admin password:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Then open: http://your-server-ip:8080

**Note:** Jenkins requires a compatible Java runtime. Check the Jenkins documentation for supported Java versions.

---

## Docker

Official docs: https://docs.docker.com/engine/install/ubuntu/

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add user to docker group (log out / in or newgrp to apply)
sudo usermod -aG docker $USER
newgrp docker
docker ps
```
If Jenkins needs Docker access:
```bash
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
Check Docker status:
```bash
sudo systemctl status docker
```

---

## Trivy (Vulnerability Scanner)

Docs: https://trivy.dev/v0.65/getting-started/installation/

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy


trivy --version
```

## Python Package Installation in the Amazon Ubuntu Ami Image [ Preinstalled 3.12 ]

```bash
# 1. Update package list
sudo apt update

# 2. Install required dependencies for adding a new Python version
sudo apt install -y software-properties-common

# 3. Add the deadsnakes PPA (Personal Package Archive) to get newer Python versions
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update

# 4. Install the specific version of Python you want
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev

curl -sS https://bootstrap.pypa.io/get-pip.py | sudo python3.10


ls /usr/bin/python3*

sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.12 2
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1


sudo update-alternatives --config python3

```
## Choose python3.10 for Jenkins pipeline.

## If your using the Plan VM
```bash
sudo apt-get update
sudo apt install -y python3.10 python3.10-venv python3.10-distutils python3.10-dev
```
---
---
## SonarQube Docker Container Run for Analysis

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v sonarqube_data:/opt/sonarqube/data \
  -v sonarqube_logs:/opt/sonarqube/logs \
  -v sonarqube_extensions:/opt/sonarqube/extensions \
  sonarqube:25.9.0.112764-community
```

---

## Jenkins Credentials to Store

| Purpose       | ID            | Type          | Notes                               |
|---------------|---------------|---------------|-------------------------------------|
| Email         | mail-cred     | Username/app password |                                  |
| SonarQube     | sonar-token   | Secret text   | From SonarQube application         |
| Docker Hub    | docker-cred   | Secret text   | From your Docker Hub profile       |

Webhook example:  
`http://<jenkins-ip>:8080/sonarqube-webhook/`
**E-mail Notification:**
- SMTP server: smtp.gmail.com
- Default user e-mail suffix: @gmail.com
- Use SMTP Authentication: Yes
- User Name: example@gmail.com
- Password: Use credentials
- Use TLS: Yes
- SMTP Port: 587
- Reply-To Address: example@gmail.com
