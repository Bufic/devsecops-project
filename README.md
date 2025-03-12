# DevSecOps Pipeline Implementation for Tic Tac Toe Game

![Screenshot 2025-03-04 at 7 16 48 PM](https://github.com/user-attachments/assets/7ed79f9c-9144-4870-accd-500085a15592)

![image](https://github.com/user-attachments/assets/5b2813a5-f493-4665-8964-77359b5be93a)

## Features

- 🎮 Fully functional Tic Tac Toe game
- 📊 Score tracking for X, O, and draws
- 📜 Game history with timestamps
- 🏆 Highlights winning combinations
- 🔄 Reset game and statistics
- 📱 Responsive design for all devices

## Technologies Used

- React 18
- TypeScript
- Tailwind CSS
- Lucide React for icons

## Project Structure

```
src/
├── components/
│   ├── Board.tsx       # Game board component
│   ├── Square.tsx      # Individual square component
│   ├── ScoreBoard.tsx  # Score tracking component
│   └── GameHistory.tsx # Game history component
├── utils/
│   └── gameLogic.ts    # Game logic utilities
├── App.tsx             # Main application component
└── main.tsx           # Entry point
```

## Game Logic

The game implements the following rules:

1. X goes first, followed by O
2. The first player to get 3 of their marks in a row (horizontally, vertically, or diagonally) wins
3. If all 9 squares are filled and no player has 3 marks in a row, the game is a draw
4. Winning combinations are highlighted
5. Game statistics are tracked and displayed

## Getting Started

### Prerequisites

- Node.js (v14 or higher)
- npm or yarn

### Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/yourusername/devsecops-demo.git
   cd devsecops-demo
   ```

2. Install dependencies:

   ```bash
   npm install
   # or
   yarn
   ```

3. Start the development server:

   ```bash
   npm run dev
   # or
   yarn dev
   ```

4. Open your browser and navigate to `http://localhost:5173`

## Building for Production

To create a production build:

```bash
npm run build
# or
yarn build
```

## The build artifacts will be stored in the `dist/` directory.

---

# Tic-Tac-Toe Deployment with Jenkins & ArgoCD

This guide explains how we automated the CI/CD pipeline for our Tic-Tac-Toe application using:

- Jenkins as the Continuous Integration (CI) tool.
- ArgoCD as the Continuous Deployment (CD) tool.
- ngrok to expose Jenkins to the internet for GitHub webhooks.
- Kubernetes for container orchestration (running locally with kind).

---

### 1. Setting Up Jenkins Webhooks with ngrok

Since Jenkins is running locally, GitHub cannot send webhooks directly to it.
To solve this, we use ngrok to create a secure public URL for Jenkins.

#### Step 1: Install ngrok

Download and install ngrok on your machine:

```
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update && sudo apt install ngrok
```

#### Step 2: Start ngrok for Jenkins

```
ngrok http 8080
```

"8080" because Jenkins runs on port 8080.

You'll see output like this:

```
Forwarding   https://random-string.ngrok.io -> http://localhost:8080
```

Copy the HTTPS URL (https://random-string.ngrok.io).

#### Step 3: Configure Webhook in GitHub

1. Go to your GitHub repository (devsecops-project).
2. Click on Settings > Webhooks.
3. Click "Add webhook".
4. Set:
   - Payload URL: Paste the ngrok URL + /github-webhook/
     (Example: https://random-string.ngrok.io/github-webhook/)

- Content type: application/json
- Secret: (Leave empty or set your own)
- Events: Select "Just push events."
- Click Add webhook.
  GitHub will now send push events to Jenkins!

---

### 2. Configuring Jenkins for GitHub Integration

#### Step 1: Install Required Jenkins Plugins

Go to Manage Jenkins > Plugin Manager > Available Plugins, then install: ✅ GitHub Integration Plugin (for webhooks).
✅ Pipeline Plugin (for writing Jenkinsfiles).
✅ Docker Pipeline Plugin (for building Docker images).
✅ Trivy Plugin (for vulnerability scanning).
✅ Git Plugin (for cloning the repo).

## After installing, restart Jenkins.

#### Step 2: Add GitHub Credentials to Jenkins

Jenkins needs GitHub access to:

- Clone the repo.
- Push changes (e.g., updating the deployment.yaml file).

1. Go to Manage Jenkins > Manage Credentials.
2. Select Global credentials and click Add Credentials.
3. Set:

- Kind: Username and password
- Username: Your GitHub username.
- Password: Your GitHub personal access token (PAT).
- ID: github-credentials (this is what i referenced in the Jenkinsfile. You can rename yours).

4. Click Save.

---

#### Step 3: Configure Jenkins Project for Webhooks

1. In Jenkins, create a New Item.
2. Select Pipeline and name it devsecops-project (Or whatever you'd like to call your pipeline).
3. Go to Build Triggers:

- ✅ Check GitHub hook trigger for GITScm polling.

4. Under Pipeline:

- Select Pipeline script from SCM.
- Set SCM to Git.
- Enter your GitHub repository URL.
- Under Credentials, select github-credentials.
- Set Branch to main.

5. Click Save.

Now, whenever you push changes to GitHub, Jenkins will automatically trigger the pipeline!

---

### 1. Jenkins as CI (Continuous Integration)

Jenkins automates the building, testing, and pushing of the application. Below is a breakdown of the pipeline, starting with the agent.

#### Step 1: Setting Up the Jenkins Agent

```
pipeline {
    agent {
        docker {
            image 'node:22-bullseye'
            args '-v /var/run/docker.sock:/var/run/docker.sock -v /home/bufic/.kube:/root/.kube --network host -u root'
        }
    }
```

Why this agent?

- We use a Docker agent instead of running Jenkins directly on the host. This ensures a consistent environment.
- The image used is node:22-bullseye, which contains the Node.js runtime needed for building and testing the app.
- Mounting volumes:
  - `-v /var/run/docker.sock:/var/run/docker.sock`: Allows the Jenkins agent to communicate with the host Docker daemon to build and push images.
  - `-v /home/bufic/.kube:/root/.kube`: Provides access to the Kubernetes config file for deploying to the cluster.
- Using `--network host`: This ensures that Jenkins can access external services like Docker, Kubernetes, and ArgoCD running on the host.
- Running as root `(-u root)`: Allows the agent to install necessary dependencies.

---

#### Step 2: Checking the Commit Author

```
stage('Check Commit Author') {
    steps {
        script {
            sh 'git config --global --add safe.directory /var/lib/jenkins/workspace/Tic-Tac-Pipeline'
            def commitAuthor = sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
            echo "Last commit author: ${commitAuthor}"
            if (commitAuthor == "Jenkins" || commitAuthor == "jenkins-bot") {
                echo "Commit made by Jenkins, skipping build..."
                currentBuild.result = 'ABORTED'
                error("Build aborted due to Jenkins commit")
            }
        }
    }
}
```

Why is this needed?

- This prevents infinite loop builds caused by Jenkins pushing changes to GitHub, which would then trigger another build.
- It checks the latest commit author. If it was made by Jenkins itself, the build is aborted.

---

#### Step 3: Setting Up the Environment

```
stage('Setup Environment') {
    steps {
        sh '''
            apt-get update && apt-get install -y docker.io curl
            curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
            curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl && mv kubectl /usr/local/bin/
            curl -Lo kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
            chmod +x kind && mv kind /usr/local/bin/
            node -v
            npm -v
            docker --version
            trivy --version
            kubectl version --client
            kind version
        '''
    }
}
```

Why this step?

- Ensures that all dependencies are installed:
  - Docker: For building and pushing images.
  - Trivy: For security scanning of Docker images.
  - Kubectl: For interacting with the Kubernetes cluster.
  - Kind: To manage a local Kubernetes cluster.
- Verifies that all installed tools are accessible.

---

#### Step 4: Installing Dependencies

```
stage('Install Dependencies') {
    steps {
        sh 'npm ci'
    }
}
```

Why this step?

- Runs `npm ci`, which installs dependencies based on `package-lock.json`, ensuring consistent package versions.

---

#### Step 5: Running Unit Tests

```
stage('Unit Testing') {
    steps {
        sh 'npm test'
    }
}
```

Why this step?

- Runs all unit tests to ensure that no existing functionality is broken.

---

#### Step 6: Static Code Analysis

```
stage('Static Code Analysis') {
    steps {
        sh 'npm run lint'
    }
}
```

Why this step?

- Checks for code quality and enforces best practices using ESLint.

---

#### Step 7: Building the Project

```
stage('Build Project') {
    steps {
        sh 'npm run build'
        archiveArtifacts artifacts: 'dist/', fingerprint: true
    }
}
```

Why this step?

- Builds the application and stores the artifacts for later use.

---

#### Step 8: Docker Build and Push

```
stage('Docker Build and Push') {
    steps {
        script {
            withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_TOKEN')]) {
                sh '''
                    echo "${GITHUB_TOKEN}" | docker login ${REGISTRY} -u ${GITHUB_USERNAME} --password-stdin
                '''
            }

            def imageTag = env.GIT_COMMIT ? "sha-${env.GIT_COMMIT}" : "build-${env.BUILD_ID}"
            def fullImageName = "${REGISTRY}/${IMAGE_NAME}:${imageTag}"

            sh """
                docker build -t ${fullImageName} .
                trivy image --ignore-unfixed --severity CRITICAL,HIGH ${fullImageName}
                docker push ${fullImageName}
                kind load docker-image ${fullImageName} --name tic-tac-cluster
            """
        }
    }
}
```

Why this step?

- Builds the Docker image for the application.
- Scans the image for security vulnerabilities using Trivy.
- Pushes the image to GitHub Container Registry.
- Loads the image into Kind, making it available for local Kubernetes deployments.

---

#### Step 9: Updating Kubernetes Deployment

```
stage('Update Kubernetes Deployment') {
    steps {
        script {
            def imageTag = env.GIT_COMMIT ? "sha-${env.GIT_COMMIT}" : "build-${env.BUILD_ID}"
            def newImage = "${REGISTRY}/${IMAGE_NAME}:${imageTag}"

            sh """
                sed -i "s|image: ${REGISTRY}/.*|image: ${newImage}|g" kubernetes/deployment.yaml
                echo "Updated deployment to use image: ${newImage}"
                grep -A 1 "image:" kubernetes/deployment.yaml
            """

            withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GITHUB_USERNAME', passwordVariable: 'GITHUB_TOKEN')]) {
                sh """
                    git config --global user.name "Jenkins"
                    git config --global user.email "jenkins@example.com"
                    git pull origin main --rebase || echo "No changes to pull"
                    git add kubernetes/deployment.yaml
                    git commit -m "Update Kubernetes deployment with new image tag: ${imageTag} [skip ci]" || echo "No changes to commit"
                    git push https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/Bufic/devsecops-project.git HEAD:main
                """
            }
        }
    }
}
```

Why this step?

- Updates the deployment.yaml file with the new Docker image.
- Pushes the changes back to GitHub to trigger ArgoCD for deployment.

---

## 2. ArgoCD as CD (Continuous Deployment)

## ArgoCD ensures that the Kubernetes cluster always runs the latest application version by monitoring the deployment.yaml file in the GitHub repository and applying changes automatically.

#### Step 1: Installing ArgoCD on the Kubernetes Cluster

To install ArgoCD, we deploy it into a dedicated argocd namespace.

##### Apply the ArgoCD Manifest

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

##### What happens here?

- Creates the argocd namespace where ArgoCD will run.
- Deploys ArgoCD components like:
  - API server
  - Repo server
  - Application controller
  - UI and CLI tools

---

#### Step 2: Exposing the ArgoCD Server

By default, ArgoCD’s UI and API are not accessible externally. We expose it using a LoadBalancer.

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

##### Why this step?

- Changes the ArgoCD service type to LoadBalancer, making it externally accessible.
- Once it gets an external IP, we can access the UI in a browser.

---

#### Step 3: Getting the ArgoCD Admin Password

ArgoCD generates a default admin password stored as a Kubernetes secret.

```
kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```

##### Why this step?

- This command extracts the encoded admin password and decodes it for use.

---

#### Step 4: Logging into the ArgoCD UI

- Open a browser and go to:

```
http://<ARGOCD_EXTERNAL_IP>
```

- Log in using:
  - Username: `admin`
  - Password: (retrieved in the previous step)

---

#### Step 5: Connecting ArgoCD to the GitHub Repository

ArgoCD needs permission to pull manifests from our GitHub repository.

##### Add the Repository:

```
argocd repo add https://github.com/your-project-repo.git \
  --username <GITHUB_USERNAME> \
  --password <GITHUB_PAT>
```

Why this step?

- This allows ArgoCD to sync the Kubernetes manifests from the GitHub repository.

---

#### Step 6: Creating an ArgoCD Application

To deploy our application using ArgoCD, we define an ArgoCD Yaml file in our kubernetes manifest folder.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tic-tac-app
  namespace: argocd # ArgoCD is installed in this namespace
spec:
  project: default
  source:
    repoURL: "https://github.com/Bufic/devsecops-project.git" # Your GitHub repo
    targetRevision: main # Branch to track
    path: kubernetes # Folder where deployment manifests are stored
  destination:
    server: https://kubernetes.default.svc # Deploy to the in-cluster Kubernetes
    namespace: default # Your application's namespace
  syncPolicy:
    automated:
      prune: true # Remove old resources not in Git
      selfHeal: true # Auto-repair drift
```

What does this do?

- Creates an ArgoCD application named tic-tac-app.
- Deploys the Kubernetes manifests from the kubernetes directory in the GitHub repository.
- `selfHeal: true` ensures that ArgoCD fixes any manual changes to the cluster.
- `prune: true` removes resources not in Git.

---

#### Step 7: Verifying Deployment in ArgoCD

1. Open the ArgoCD UI at `http://<ARGOCD_EXTERNAL_IP>`.
2. Click on tic-tac-app.
3. You should see:
   - Pods running
   - Deployment status
   - Synced state (green checkmark)

---

#### Step 8: Testing the Application

Get the external IP of the application service:

```
kubectl get svc -n default
```

Look for the `LoadBalancer` service and access it in a browser:

```
http://<APP_EXTERNAL_IP>
```

If everything works, your application is successfully deployed using ArgoCD!

---

### Final Thoughts

Jenkins automates building and pushing images.
ArgoCD pulls and deploys Kubernetes manifests.
Any change to deployment.yaml in GitHub triggers an automatic deployment via ArgoCD.
Your CI/CD pipeline is now fully automated!!
