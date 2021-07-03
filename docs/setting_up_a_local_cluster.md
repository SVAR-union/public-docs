# Setting Up a Local Development Cluster
This document describes how to set up a local Kubernetes cluster for
development.  

## Prerequisites
Before following this tutorial, please make sure you have the necessary
prerequisites installed. It's also strongly recommended to follow the tutorial
using a clean installation of Ubuntu 20.04 LTS.

To set up a local cluster on a compatible platform your will need:

- A working installation of [Git](https://git-scm.org/).
- [Docker Engine for Linux](https://docs.docker.com/engine/install/ubuntu/) or
  [Docker Desktop for other platforms](https://www.docker.com/get-started), a
  container runtime.
- [Kind](https://kind.sigs.k8s.io/), a local cluster provisioner.
- [Kubectl](https://kubernetes.io/docs/tasks/tools/installing-kubectl-linux), a
  tool for controlling clusters.
- [Helm](https://helm.sh/), a package manager for Kubernetes.
- [Skaffold](https://skaffold.dev), a tool for local Kubernetes development.

## Installing prerequisites

The following instructions were carried out on a fresh installation of Ubuntu
20.04 LTS under KVM and were able to reproduce the cluster as of 2020-07-03.

### Git

To install Git, run:
```
sudo apt install git
```

### Docker

To install Docker, follow the [offical
guide](https://docs.docker.com/engine/install/ubuntu/). We are going to copy
the guide as-is.

#### Installing Docker itself

First, ensure that previous versions of Docker are removed. To do this, run:
```
sudo apt-get remove docker docker-engine docker.io containerd runc
```

When any leftovers are removed, install the necessary packages via the package
manager:
```
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

When Docker prerequisites are satisfied, add the Docker repository key to your
keyring:
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

After adding the keyring, add the reposiotry itself to the list of
repositories:
```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

When the repository has been added to the list of sources, install the Docker
packages:
```
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

#### Post-installation steps
After Docker has been installed, it should be able to run without root. To do
that, we need to create a group for Docker:
```
sudo groupadd docker
```

Once the group has been created, add the current user to it:
```
sudo usermod -aG docker $USER
```
Now you should be a member of the `docker` group.

Next, to be able to run docker, you should apply the group changes. To do that,
run:
```
newgrp docker
```
After the command finishes, group changes should be applied and the current
user should be able to run Docker.

To check if you're able to run Docker without sudo, run:
```
docker run hello-world
```
The following steps assume that your user account is able to run Docker without
sudo, so it is imperative you ensure that the command above finished
successfully.

### Go

Most of the infrastructure-related tools require Go binaries to run. To setup
Go, first download the binaries:
```
curl -LO https://golang.org/dl/go1.16.5.linux-amd64.tar.gz
```

Then, clean up any previous installations of Go:
```
sudo rm -rf /usr/local/go
```

Once any leftovers have been cleaned up, unpack the downloaded binaries. Since
the unpacking command writes to a path that required elevated privileges, make
sure to run it with `sudo`.
```
sudo tar -C /usr/local -xzf go1.16.5.linux-amd64.tar.gz
```

Once you've unpacked the binaries, make sure to add them to `PATH`. On Linux,
the most widely supported option is to add the command to add Go binaries to
the `PATH` environment variable to your `.profile` file. Please be aware that
the changes will only be applied after you re-login into your account. So, to
add Go binaries to `PATH`, run:
```
printf "\n# Add GOROOT to PATH\n%s\n" "export PATH=\$PATH:/usr/local/go/bin" >> ~/.profile
```

Then, log out of your user session by clicking the _Power Off → Log Out / Shut
Down → Log Out → Log Out_ button. Or you could reboot.

Whatever path you choose, log into your session again. Once you've logged into
your session, ensure that Go binaries are in your `PATH`:
```
go --version
```
The correct output should look like:
```
go version go1.16.4 linux/amd64
```

Once you've added Go binaries to `PATH`, you need to add the path to Go modules
to your `PATH`:
```
printf "\n# Add GOPATH to PATH\n%s\n" "export PATH=\$PATH:`(go env GOPATH)`/bin" >> ~/.profile
```
Again, either relogin or reboot to ensure that the OS applied all changes to
`PATH`. After that, your Go installation should be set up.

### Kind
To install kind, run the following command:
```
GO111MODULE="on" go get sigs.k8s.io/kind@v0.11.1
```
Then, check if it is accessible:
```
kind --version
```
The correct output would look like:
```
kind version 0.10.0
```
Now, kind is installed on your system.

### Kubectl
To install kubectl, first download the latest release:
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

Once the release has been downloaded, install it:
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Check if it's installed:
```
kubectl version --client
```
The correct output should look like:
```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.2", GitCommit:"092fbfbf53427de67cac1e9fa54aaa09a28371d7", GitTreeState:"clean", BuildDate:"2021-06-16T12:59:11Z", GoVersion:"go1.16.5", Compiler:"gc", Platform:"linux/amd64"}
```
Kubectl should be installed now.

### Helm
To install Helm, the easiest way is to use Snap:
```
sudo snap install helm --classic
```

Check if Helm is installed:
```
helm --version
```
The correct output should look like:
```
version.BuildInfo{Version:"v3.5.4", GitCommit:"1b5edb69df3d3a08df77c9902dc17af864ff05d1", GitTreeState:"clean", GoVersion:"go1.15.11"}
```
Helm should be installed now.

### Skaffold
To install Skaffold, download its binaries and install them by running the
following commands:
```
curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64 && \
sudo install skaffold /usr/local/bin/
```

Check if Skaffold has been installed:
```
skaffold version
```
The output should look like:
```
v1.24.1
```
Now, Skaffold should be ready, and so are your other prerequisites. You should
be able to provision a local cluster.

## Ensuring access to the Git repos

Current Skaffold manifests use remote Git repos to resolve its dependencies.
Therefore, you need to have access to the repositories and to be able to access
the repos from your operating system. That means that you should have proper
SSH keys set up and integrated with Git.

To do that, first make sure that you have your SSH keys. If not, generate new
ones via:
```
ssh-keygen -t ed25519
```
The command will ask you questions. You could either make your own decisions or
just press Enter until the command is done.

After the command is done, you should have your SSH keys ready. Next, you need to make sure that your SSH agent is running. To do this, run:
```
eval "$(ssh-agent -s)"
```
The expected output is:
```
Agent pid 23594
```
The pid will probably be different for you.

Next, when your agent is running, add your SSH keys to it:
```
ssh-add ~/.ssh/id_ed25519
```
The command should notify you that a new identity has been added.

Now, when you've created the SSH keys, run the agent, added an identity, you
should add your keys to your Github account. For that, you'll need to log into
your account and go to _Settings → SSH and GPG keys → New SSH key_. Github will
ask you to enter your key.

Go back to your terminal session and copy your _public_ key to clipboard:
```
xclip -selection clipboard < ~/.ssh/id_ed25519.pub
```
Once the command completes, your public key should be in your clipboard.

Go back to Github, the page that asks you to enter the key. Paste your key and
press _Add SSH key_. Now, your key should be added to Github, and you are ready
to deploy the project locally.

## Deploying the project
### Setting up the environment
Once the prerequisites are installed, to run a project you'll need a local
development cluster. To provision the cluster, run the following command:
```
kind create cluster
```

Kind will set up the cluster for you and notify you when it's ready:
```
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼 
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```
Once the cluster is set up, the environment is set up, and you are ready to
deploy services.

### Running services
To run any service, you'll first have to clone it. You should clone to a convenient location, so it's best to create it first:
```
mkdir ~/Documents/svar-achievements/
```

Navigate to the newly created directory:
```
cd ~/Documents/svar-achievements
```

Then, clone the service you want to run. For example, the internationalization one:
```
git clone git@github.com:SVAR-union/service_i18n
```

Once Git has finished cloning the service, go the the service's directory:
```
cd service_i18n
```
When you're inside the directory, you can deploy the service by running:
```
skaffold dev
```
Skaffold will pull, build, tag and stabilize your deployments. It will also
watch for any changes you make to the source code, so it can rebuild your
deployments and make it available for you to access and test.
