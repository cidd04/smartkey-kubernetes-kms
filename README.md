# SmartKey Kubernetes KMS Plugin


Prerequisite:
1. GoLang must be installed in server you want to build project/create installer (not required for running installer directly).
2. We need existing Kubernetes cluster to configure and run "SmartKey Kubernetes KMS Plugin".
3. To build/run code or create installer, git pull this project inside your go directory (eg: ~/go/src/)


This project allow to use SmartKey as KMS provider for Kubernetes. Refer [link](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/) to understand how kubernetes work with external KMS.


#### Sample "smartkey-grpc.conf" file
    {
      "smartkeyApiKey": "<smartkey-api-key>",
	  "encryptionKeyUuid": "<uuid-for-aes-encryption-key-in-SmartKey>",
	  "iv": "<Initialization vector for your AES algo as per your key size (128, 192, 256)>",
	  "socketFile": "<path-to-your-sock-file>",
	  "smartkeyURL": "<smartkey-url>"
	}

### Getting Started (Developer)
Take checkout of current project.

##### Command to test, build or clean project (from inside project). 
This is only for developers who want to build binary of grpc server manually or execute unit test cases.

    - Run below command to test code
        $ make test
    - Run below command to build code
        $ make build
    - Run below command to clean workspace
        $ make clean

##### Build command will generate smartkey-kms binary 
Below steps are for developers who want to run gRPC server directly without using installer (explained later in documentation).
  - Execute below command to run binary. In next sections we will see how we can create installer and use that to run plugin.
    - make build (to build your code and get smartkey-kms)
    - sudo ./smartkey-kms --socketFile <sock-file-path> --config <config-file->
        - sock-file-path:  Path where you want to create your unix socket file eg: /etc/smartkey/smartkey.socket
        - config-file: Path to your config file. eg: conf/smartkey-grpc.conf
        - Note: "sock-file-path" should exist(eg /etc/smartkey). If not, please create before running server.

### Creating Debian installer from plugin binary
  - Install below tools
    - sudo apt-get install dh-make
    - sudo apt-get install devscripts build-essential lintian
 - Execute script "create_installer.sh". It will create deb file in one level above your PWD

## Installation
Instead of manually building code (as mentioned in Getting started 'Developer' section), we can directly install plugin using precomplied .deb file created in previous step "Creating Debian installer from plugin binary".

#### Installing and configuring Plugin from Debian installer
  - Execute below command to install package
    - sudo dpkg -i smartkey-kmsplugin_1.0-1_amd64.deb
  - Above command only install plugin binary in your machine but don't start it.
  - Note: We need to do configure our config files before starting service.
  - Update configuration at "/etc/smartkey/smartkey-grpc.conf". Refer above section for more details.
  - Execute below command to run plugin gRPC server 
    - sudo service smartkey-grpc start &
  - Verify if service started correctly using below command
    - sudo service smartkey-grpc status

## Configure api-server to use "SmartKey Kubernetes KMS Plugin"
For both, 1) running plugin manually or 2) using installer and "smartkey-grpc" service created by installer to run it, we need to perform below steps.

Open "/etc/kubernetes/manifests/kube-apiserver.yaml" file and add below configuration.

    - spec.containers.command.kube-apiserver
        --encryption-provider-config=/etc/smartkey/smartkey.yaml

    The installer will automatically create "smartkey.yaml" and copy to the desired (/etc/smartkey/) location for you with prepopulated configuration.

    Add "volumemount" and "volume host path" to kube-apiserver.yaml. 
    Without below configuration, api-server won't be able to read smartkey.yaml and smartkey.sock files.


    =====
    volumeMounts:
       - mountPath: /etc/smartkey
           name: smartkey-kms
           readOnly: true
    ...
    ...
    volumes:
       - hostPath:
           path: /etc/smartkey
           type: DirectoryOrCreate
         name: smartkey-kms
    ====
    **Installer will not add above configurations and need to be added manually.


Save "kube-apiserver.yaml" and exit. Apiserver will detect changes in "kube-apiserver.yaml" file and restart.
    
 ## Verify if plugin is configured correctly
    - Create new secret and you should be able to see logs in plugin service logs. 
    - User below command to see logs
        -   journalctl -xe
    - Command to create new secret which will be encrypted by our plugin.
        -   kubectl create secret generic install3 -n default --from-literal=es-engkey=equinixdata
    - Command to encrypt all of your existing secrets.
        -   kubectl get secrets --all-namespaces -o json | kubectl replace -f -
    
## Troubleshooting
    1. Permission denied error while trying to encrypt old secrets using below command
        Create new secret and you should be able to see logs in plugin service logs. 
        $ kubectl get secrets --all-namespaces -o json | kubectl replace -f -
        Fix: Make sure that kubernetes config directory has the same permissions as kubernetes config file.
            $ mkdir -p $HOME/.kube
            $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
            $ sudo chown $(id -u):$(id -g) $HOME/.kube/config 
            Add change permissions on $HOME/.kube/ directory.
                $ sudo chown $(id -u):$(id -g) $HOME/.kube/
    2. For error "go get: no install location for directory" execute below commands.
        export GOBIN=$HOME/work/bin
        export GOPATH=$HOME/go
    3. In order to uninstall debian package use below command.
        sudo dpkg -P smartkey-kmsplugin

## Support email
    ES-ENG-SECURITY <ES-ENG-SECURITY@equinix.com>