CI/CD Pipeline Implementation: Jenkins, Ansible, Artifactory, SonarQube, & PHP
This repository houses the configuration and code for a complete Continuous Integration and Continuous Delivery (CI/CD) pipeline for a PHP-based TODO web application. The pipeline automates code checkout, dependency management, unit testing, static code analysis, artifact packaging, and multi-environment deployment.

Architecture & Environments
The infrastructure relies on Nginx as a reverse proxy to serve the applications and tools. The deployment targets multiple environments mapped to dedicated virtual instances:

CI Environment: Jenkins (CI Server), SonarQube (Code Quality), Artifactory (Artifact Repository), and a backend Database.

Deployment Environments: Dev, SIT (System Integration Testing), UAT (User Acceptance Testing), Pentest, Preprod, and Prod. Each environment runs the Tooling App and TODO WebApp connected to a Database.

1. Infrastructure Preparation
Provision standard Linux EC2 virtual instances for the required environments.

DNS Configuration:
Create the following DNS records pointing to the respective server IP addresses (assuming a base domain of steghub.com):

[https://ci.infradev.steghub.com](https://ci.infradev.steghub.com) -> Jenkins

[https://sonar.infradev.steghub.com](https://sonar.infradev.steghub.com) -> SonarQube

[https://artifacts.infradev.steghub.com](https://artifacts.infradev.steghub.com) -> Artifactory

https://todo.<environment>.steghub.com -> TODO WebApp across Dev, SIT, UAT, Pentest, Pre-Prod, and Prod

https://tooling.<environment>.steghub.com -> Tooling App across all environments


Ansible Inventory Setup:
Define the environment inventories dynamically or statically. Example for dev:

[tooling]
<Tooling-Web-Server-Private-IP-Address>

[todo]
<Todo-Web-Server-Private-IP-Address>

[nginx]
<Nginx-Private-IP-Address>

[db:vars]
ansible_user=ec2-user
ansible_python_interpreter=/usr/bin/python

[db]
<DB-Server-Private-IP-Address>

2. Jenkins & GitHub Integration

   1-Install Dependencies: Install Jenkins, PHP, and Composer on the CI server.

   sudo apt install -y zip libapache2-mod-php phploc php-{xml,bcmath,bz2,intl,gd,mbstring,mysql,zip}

   
   2- Install Jenkins Plugins: Navigate to Jenkins UI and install the Blue Ocean, Plot, Artifactory, and SonarScanner plugins.

   
   3- GitHub Authentication:

      -Generate a Personal Access Token in GitHub under Developer Settings.

      -In Jenkins (Blue Ocean), create a new multibranch pipeline, select GitHub, and input the token to connect the repository.

4. Database Initialization
  On the target database server, create the necessary database and user for the PHP TODO application:

    CREATE DATABASE homestead;
    CREATE USER 'homestead'@'%' IDENTIFIED BY 'sePret^i';
    GRANT ALL PRIVILEGES ON * . * TO 'homestead'@'%';

     Update the .env.sample file in the application repository with these connectivity details

5. SonarQube & PostgreSQL Configuration (Ubuntu 20.04)
   Kernel Tuning:
   SonarQube requires kernel parameter adjustments for optimal performance. Update these permanently in /etc/security/limits.conf:

   sonarqube   -   nofile   65536
   sonarqube   -   nproc    4096

Apply session changes:

  sudo sysctl -w vm.max_map_count=262144
  sudo sysctl -w fs.file-max=65536
  ulimit -n 65536
  ulimit -u 4096

  Install Java and PostgreSQL 10:

  sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install wget unzip openjdk-11-jdk openjdk-11-jre -y
sudo update-alternatives --config java

# Install PostgreSQL
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt-get -y install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

Configure PostgreSQL for SonarQube:

su - postgres
createuser sonar
psql
ALTER USER sonar WITH ENCRYPTED password 'sonar';
CREATE DATABASE sonarqube OWNER sonar;
grant all privileges on DATABASE sonarqube to sonar;
\q
exit

Install SonarQube 7.9.3:

cd /tmp && sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-7.9.3.zip
sudo unzip sonarqube-7.9.3.zip -d /opt
sudo mv /opt/sonarqube-7.9.3 /opt/sonarqube
sudo groupadd sonar
sudo useradd -c "user to run SonarQube" -d /opt/sonarqube -g sonar sonar 
sudo chown sonar:sonar /opt/sonarqube -R


Configure SonarQube Database Properties:
Edit /opt/sonarqube/conf/sonar.properties:


Properties
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube


Edit /opt/sonarqube/bin/linux-x86-64/sonar.sh to set RUN_AS_USER=sonar.

Create Systemd Service:
Create /etc/systemd/system/sonar.service:

[Unit]
Description=SonarQube service
After=syslog.target network.target
[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096
[Install]
WantedBy=multi-user.target

Start the service:

sudo systemctl start sonar
sudo systemctl enable sonar


Access SonarQube at http://<server_IP>:9000 (Default credentials: admin / admin). Navigate to Administration > Configuration > Webhooks and create a webhook pointing to http://{JENKINS_HOST}/sonarqube-webhook/.

5. Jenkinsfile Pipeline Stages
Create a Jenkinsfile in the root of the repository to define the automated workflow. Ensure the Jenkins sonar-scanner.properties tool configuration includes the sonar.projectKey=php-todo and points to the SonarQube server URL.

pipeline {
    agent any

    environment {
        PHP_BIN = '/usr/bin/php'
        DB_USER = 'homestead'
        DB_PASS = 'sePret^i'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/StegTechHub/php-todo.git'
            }
        }

        stage('Prepare Dependencies') {
            steps {
                sh '''
                    echo "APP_NAME=Laravel" > .env
                    echo "APP_ENV=local" >> .env
                    echo "APP_KEY=base64:randomstring1234567890=" >> .env
                    echo "APP_DEBUG=true" >> .env
                    echo "APP_URL=http://localhost" >> .env
                    echo "DB_CONNECTION=mysql" >> .env
                    echo "DB_HOST=127.0.0.1" >> .env
                    echo "DB_PORT=3306" >> .env
                    echo "DB_DATABASE=homestead" >> .env
                    echo "DB_USERNAME=${DB_USER}" >> .env
                    echo "DB_PASSWORD=${DB_PASS}" >> .env
                    
                    mkdir -p bootstrap/cache storage/framework/views storage/framework/sessions storage/framework/cache
                '''
                
                sh 'COMPOSER_ALLOW_SUPERUSER=1 composer install --ignore-platform-reqs --no-scripts --no-plugins'
                
                sh '''
                    # Target the active system PHP binary (PHP 7.2.34)
                    PHP_BIN=""
                    for candidate in /usr/bin/php /usr/bin/php7.2 /usr/local/bin/php7.2 $(which php 2>/dev/null); do
                        if [ -x "$candidate" ]; then
                            PHP_BIN="$candidate"
                            break
                        fi
                    done
                    
                    if [ -z "$PHP_BIN" ]; then
                        echo "--------------------------------------------------------"
                        echo "ERROR: Compatible PHP binary was not found!"
                        echo "--------------------------------------------------------"
                        exit 1
                    fi
                    
                    echo "Using PHP binary: $PHP_BIN"
                    $PHP_BIN -v
                    
                    $PHP_BIN artisan key:generate
                    DB_USERNAME=${DB_USER} DB_PASSWORD=${DB_PASS} $PHP_BIN artisan migrate --force
                    DB_USERNAME=${DB_USER} DB_PASSWORD=${DB_PASS} $PHP_BIN artisan db:seed --force
                '''
            }
        }

        stage('Execute Unit Tests') {
            steps {
                sh '''
                    PHP_BIN="/usr/bin/php"
                    DB_CONNECTION=mysql DB_HOST=127.0.0.1 DB_PORT=3306 DB_DATABASE=homestead DB_USERNAME=''' + env.DB_USER + ''' DB_PASSWORD=''' + env.DB_PASS + ''' $PHP_BIN ./vendor/bin/phpunit
                '''
            }
        }

        //stage('Code Analysis') {
          //  steps {
            //    sh 'mkdir -p build/logs'
              //  sh '/usr/bin/php /usr/bin/phploc app/ --log-csv build/logs/phploc.csv'
            //}
        //}

        stage('SonarQube Quality Gate') {
            when { 
                branch pattern: '^(develop|hotfix|release|main).*$', comparator: 'REGEXP'
            }
            environment {
                scannerHome = tool 'SonarQubeScanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "${scannerHome}/bin/sonar-scanner -Dproject.settings=sonar-project.properties"
                }
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package Artifact') {
            steps {
                sh 'zip -qr php-todo.zip . -x "*.git*" -x "vendor/*" -x "node_modules/*"'
            }
        }

        stage('Archive Artifact in Jenkins') {
            steps {
                archiveArtifacts artifacts: 'php-todo.zip', fingerprint: true
            }
        }
        
        stage('Deploy to Environments') {
            steps {
                script {
                    def environments = ['dev']
                    for (String targetEnv : environments) {
                        echo "Triggering deployment for environment: ${targetEnv}"
                        build job: 'Deploy-To-Server', 
                            parameters: [[$class: 'StringParameterValue', name: 'env', value: targetEnv]], 
                            propagate: false, 
                            wait: true
                    }
                }
            }
        }
    }
}


6. Final Operational Tasks
Add 2 servers as Jenkins slave nodes and configure Jenkins to distribute pipeline jobs across them.

7. woraround to fix

   successful DevSecOps pipeline and infrastructure configuration for Project 14.


    -End-to-End Pipeline Success: Build #124 of NewPipelineSept16 completed all stages seamlessly, moving from SCM checkout through the SonarQube Quality Gate, artifact packaging, and environment deployment. Historical runs (like build #75) successfully executed code analysis and plotted the PHPLOC coverage reports.

    -SonarCloud Integration: Jenkins is correctly configured to communicate with SonarCloud ([https://sonarcloud.io](https://sonarcloud.io)). The Project14-Ansible-Jenkins repository successfully passed the SonarQube      Quality Gate, registering 4,000 lines of code and earning "A" ratings for Maintainability.

    -EC2 Node Provisioning: Command-line configurations on your EC2 instance show successful MariaDB user privilege assignments for the homestead database, alongside package installations for zip and the phploc executable.

     -Agent Connectivity: The Jenkins Nodes management dashboard confirms that agent2 and Web1-UAT-project14-jen-agent-1 are both online, synchronized, and actively handling Linux (amd64) workloads.   Since the pipeline  is fully operational and the quality gates are passing, where would you like to focus next? We could refine the Ansible playbooks used to provision these database and web nodes, document the deployment architecture for your portfolio, or set up automated notifications for the pipeline status.

Configure a GitHub Webhook pointing to the Jenkins server to automatically trigger the pipeline upon code pushes.

Use the parametrized env variable in the Jenkins UI to deploy the artifact across the defined integration and production environments.


DevSecOps architecture for Project 14 based on all the provided configurations and pipeline executions.

1. Cloud Infrastructure Tier (AWS EC2)
The environment runs on a distributed AWS EC2 architecture using t3.small instances.   


Jenkins Controller / Ansible Control Node: Hosted on the Project 11 Jenkins-Ansible instance in the us-east-1d availability zone (IP: 3.85.220.247). This node orchestrates the builds and holds the Ansible playbooks.   


UAT Deployment Agents: Two target agent nodes are provisioned in us-east-1a:

Web1-UAT project14 jen agent 1 (IP: 18.207.183.195).   


Web2-UAT project14 jen agent 2 (IP: 44.200.241.192).   


Agent Connectivity: Both UAT web servers are successfully connected to the Jenkins controller as Linux (amd64) execution nodes (agent2 and Web1-UAT-project14-jen-agent-1).   


2. CI/CD Pipeline Orchestration
The deployment process is split into an upstream build pipeline and a downstream deployment pipeline, configured as Infrastructure as Code via a Jenkinsfile.   

Upstream Pipeline (NewPipelineSept16): Handles the Continuous Integration phase. The stages execute sequentially: Checkout SCM (from GitHub) → Prepare Dependencies (dynamically generating the Laravel .env file) → Execute Unit Tests → SonarQube Quality Gate → Plot Code Coverage Report (using phploc) → Package Artifact (creating php-todo.zip) → Archive Artifact → Deploy to Environments.   


Downstream Pipeline (Deploy-To-Server): Triggered automatically upon the successful completion of the upstream build. This pipeline executes the final release to the development/UAT servers.   


3. DevSecOps & Quality Assurance
Security and code quality are integrated directly into the CI loop to prevent regressions.

SonarQube / SonarCloud: Jenkins is integrated with [https://sonarcloud.io](https://sonarcloud.io) using an authentication token. The SonarQube Scanner tool is configured to install automatically. The Project14-Ansible-Jenkins project is actively analyzed, currently passing the Quality Gate with 4,000 lines of code and an "A" rating for Maintainability. Developers also use the SonarQube IDE extension within Visual Studio Code for local linting.   


Code Coverage: phploc is installed on the agent nodes to analyze lines of code and structure, which Jenkins plots into visual trend graphs during the build.   

4. Artifact Management

JFrog Artifactory: Configured centrally in Jenkins (http://localhost:8081/artifactory) to manage and store binary deliverables and deployment packages.   

5. Application & Database Tier
Application Stack: A PHP/Laravel web application (php-todo).   

Database Infrastructure: Backed by a MariaDB/MySQL database. The target servers are configured with a homestead database and user, with full privileges granted for local loopback connections.   


Configuration Management: Ansible is used to enforce state on the servers, with a structured directory (roles, playbooks, inventory) maintained alongside the application code in Visual Studio Code. Server package prerequisites, such as zip and curl, are managed at the OS level.   


<img width="530" height="747" alt="image" src="https://github.com/user-attachments/assets/0db44dd1-3fff-4ba1-903c-c429c68a6aed" />








 
