# Jenkins & SonarQube Integration Guide

---

## 1. Generate Auth Token in SonarQube

1. Open SonarQube UI: `http://<SONARQUBE_IP>:9000`
2. Go to: **User Profile (top right)** $\rightarrow$ **My Account** $\rightarrow$ **Security**
3. Under **Generate Tokens**:
   - **Name:** `jenkins-token`
   - **Type:** `Global Analysis Token`
4. Click **Generate** and copy the token string immediately.

---

## 2. Install Plugin in Jenkins

1. Open Jenkins UI: `http://<JENKINS_IP>:8080`
2. Navigate to: **Manage Jenkins** $\rightarrow$ **Plugins** $\rightarrow$ **Available plugins**
3. Search and install: **SonarQube Scanner for Jenkins**
4. Restart Jenkins once installation completes.

---

## 3. Add SonarQube Token to Jenkins Credentials

1. Go to: **Manage Jenkins** $\rightarrow$ **Credentials** $\rightarrow$ **System** $\rightarrow$ **Global credentials (unrestricted)**
2. Click **Add Credentials**:
   - **Kind:** `Secret text`
   - **Secret:** `<PASTE_SONARQUBE_TOKEN>`
   - **ID:** `sonar-token`
   - **Description:** `SonarQube Auth Token`
3. Click **Create**.

---

## 4. Configure Server & Scanner in Jenkins

### A. Server Link
1. Navigate to: **Manage Jenkins** $\rightarrow$ **System** $\rightarrow$ **SonarQube servers**
2. Check the box: **Environment variables**
3. Click **Add SonarQube**:
   - **Name:** `SonarQube-Server`
   - **Server URL:** `http://<SONARQUBE_IP>:9000`
   - **Server authentication token:** Select `sonar-token`
4. Click **Apply** & **Save**.

### B. Scanner Tool
1. Navigate to: **Manage Jenkins** $\rightarrow$ **Tools** $\rightarrow$ **SonarQube Scanner installations**
2. Click **Add SonarQube Scanner**:
   - **Name:** `Sonar-Scanner`
   - Check: **Install automatically**
3. Click **Save**.

---

## 5. Configure Quality Gate Webhook in SonarQube

1. Open SonarQube UI: **Administration** $\rightarrow$ **Configuration** $\rightarrow$ **Webhooks**
2. Click **Create**:
   - **Name:** `Jenkins-Webhook`
   - **URL:** `http://<JENKINS_IP>:8080/sonarqube-webhook/`
   
   > **Note:** Ensure the trailing slash `/` is included in the URL.

3. Click **Create**.

---

## 6. Jenkins Declarative Pipeline (Jenkinsfile)

```groovy
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'Sonar-Scanner'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: '[https://github.com/your-repo/sample-app.git](https://github.com/your-repo/sample-app.git)'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh """
                        \${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=my-sample-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=\${SONAR_HOST_URL}
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}
