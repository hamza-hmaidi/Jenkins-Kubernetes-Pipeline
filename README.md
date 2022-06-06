# Create A CI/CD Pipeline With Kubernetes and Jenkins

In this lab, we are building a continuous delivery (CD) pipeline. 

We are using a very simple application written in Go. For the sake of simplicity, we are going to run only one type of test against the code.

### Prerequisites

- A running Jenkins instance
- Docker hub image registry
- for my example I’am using a preconfigured simple Kubernetes cluster that’s hosted on Google Cloud
- Ansible installed

### Jenkins Plugins to be installed

- git.
- pipeline.
- CloudBees Docker Build and Publish.
- GitHub.

# Steps:

### **Install Jenkins, Ansible, and Docker**

```bash
sudo apt update && sudo apt install -y python3 && sudo apt install -y python3-pip && sudo pip3 install ansible && sudo pip3 install openshift
echo "export PATH=$PATH:~/.local/bin" >> ~/.bashrc && . ~/.bashrc
ansible-galaxy install geerlingguy.jenkins
ansible-galaxy install geerlingguy.docker
```

**Configuring Jenkins User to Connect to the Cluster**

```bash
$ sudo cp ~/.kube/config ~jenkins/.kube/
$ sudo chown -R jenkins: ~jenkins/.kube/
```

### **Create the Jenkins Pipeline Job**

### **Create the JenkinsFile**

```bash
pipeline {
   agent any
   environment {
       registry = "magalixcorp/k8scicd"
       GOCACHE = "/tmp"
   }
   stages {
       stage('Build') {
           agent {
               docker {
                   image 'golang'
               }
           }
           steps {
               // Create our project directory.
               sh 'cd ${GOPATH}/src'
               sh 'mkdir -p ${GOPATH}/src/hello-world'
               // Copy all files in our Jenkins workspace to our project directory.               
               sh 'cp -r ${WORKSPACE}/* ${GOPATH}/src/hello-world'
               // Build the app.
               sh 'go build'              
           }    
       }
       stage('Test') {
           agent {
               docker {
                   image 'golang'
               }
           }
           steps {                
               // Create our project directory.
               sh 'cd ${GOPATH}/src'
               sh 'mkdir -p ${GOPATH}/src/hello-world'
               // Copy all files in our Jenkins workspace to our project directory.               
               sh 'cp -r ${WORKSPACE}/* ${GOPATH}/src/hello-world'
               // Remove cached test results.
               sh 'go clean -cache'
               // Run Unit Tests.
               sh 'go test ./... -v -short'           
           }
       }
       stage('Publish') {
           environment {
               registryCredential = 'dockerhub'
           }
           steps{
               script {
                   def appimage = docker.build registry + ":$BUILD_NUMBER"
                   docker.withRegistry( '', registryCredential ) {
                       appimage.push()
                       appimage.push('latest')
                   }
               }
           }
       }
       stage ('Deploy') {
           steps {
               script{
                   def image_id = registry + ":$BUILD_NUMBER"
                   sh "ansible-playbook  playbook.yml --extra-vars \"image_id=${image_id}\""
               }
           }
       }
   }
}
```

the pipeline contains four stages:

1. Build is where we build the Go binary and ensure that nothing is wrong in the build process.
2. Test is where we apply a simple  test to ensure that the application works as expected.
3. Publish, where the Docker image is built and pushed to the registry. After that, any environment can make use of it.
4. Deploy, this is the final step where Ansible is invoked to contact Kubernetes and apply the definition files.

# Test the workflow

After triggering the workflow with pushing to master:

you can  **Get the node IP address:**

```bash
kubectl get nodes -o wide
NAME                                          STATUS   ROLES    AGE   VERSION          INTERNAL-IP   EXTERNAL-IP     OS-IMAGE                             KERNEL-VERSION   CONTAINER-RUNTIME
gke-security-lab-default-pool-46f98c95-qsdj   Ready       7d    v1.13.11-gke.9   10.128.0.59   35.193.211.74   Container-Optimized OS from Google   4.14.145+        docker://18.9.7
```

sending request

```bash
$ curl 35.193.211.74:32000
{"message": "hello world"}
```