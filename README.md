# Jenkins + Terraform + Consul
Use **Jenkins** with **Terraform** for automation and **Consul** as backend service for **state**
and **configuration** files management (**applications** and **networking** stacks).

## Requirements ##
* AWS **deep-dive** profile with `admin` access to your account
* Unix/MacOS/Windows
* Terraform
* Consul
* Docker

## Prerequisites ##

### Terraform configuration ###
1. Configure an AWS profile with proper credentials:
    ```
    aws configure --profile deep-dive
    export AWS_PROFILE=deep-dive
    ```
2. Deploy the current environment:
    ```
    terraform init
    terraform validate
    terraform plan -out base.tfplan
    terraform apply "base.tfplan"
    ```
3. View all the **Terraform** resources:
    ```
    terraform state list
    ```
5. Look at a specific resource:
    ```
    terraform state show "module.vpc.aws_vpc.this[0]"
    ```
7. View all the **state** data:
    ```
    terraform state pull
    ```
9. Destroy the **current** environment (optional):
    ```
    terraform destroy -auto-approve
    ```

### Consul setup and configuration ###
1. Download the **Consul** executable from https://www.consul.io/downloads
2. Go to the **consul** directory:
    ```
    cd consul
    ```
4. Create a **data** directory:
   ```
   mkdir data
   ```
6. Launch **Consul** server instance:
    ```
    consul agent -bootstrap -config-file="config/consul-config.hcl" -bind="127.0.0.1"
    ```
7. Open a separate terminal window and generate **bootstrap token** (make note of the root **SecretID** for later):
    ```
    consul acl bootstrap
    ```
9. Check **Consul** UI (user **root** token for authentication):
    ```
    http://127.0.0.1:8500/ui/dc1/services
    ```
10. Set **CONSUL_HTTP_TOKEN** to the root **SecretID**:
    ```
    export CONSUL_HTTP_TOKEN=SECRETID_VALUE
    ```
11. Configure **Consul** using **Terraform** (set up paths, policies, and user tokens):
    ```
    terraform init
    terraform plan -out consul.tfplan
    terraform apply consul.tfplan  # make note of `SecretID's` from the output for later
    ```
12. Get token values for **Mary** and **Sally** (make note of **SecretID's** for later):
    ```
    consul acl token read -id "ACCESSOR_ID_MARY"
    consul acl token read -id "ACCESSOR_ID_SALLY"
    ```
13. Go back to the main directory: `cd ..`
14. Set the **Consul** token to **Mary** (replace SECRETID_VALUE with **Mary** secret ID): `export CONSUL_HTTP_TOKEN=SECRETID_VALUE`
15. Copy consul **backend.tf** from **backend_consul** directory: `cp backend_consul/backend.tf .`
16. Initialize and apply the backend config (enter **yes** to migrate existing state to the new backend):
    ```
    export AWS_PROFILE=deep-dive
    terraform init -backend-config="path=networking/state/globo-primary"
    terraform plan -out plan.tfplan
    terraform apply plan.tfplan
    ```
17. Check that state file has been migrated to the **Consul**: `http://127.0.0.1:8500/ui/dc1/kv/networking/state/globo-primary/edit`
18. Add the **networking** stack to **consul**:
    1. Change to **Mary's** token for **Consul**: `export CONSUL_HTTP_TOKEN=SECRETID_VALUE`
    2. Write the configuration data for **globo-primary** config:
        ```
        cd consul/data_configs
        consul kv put networking/configuration/globo-primary/net_info @globo-primary.json
        consul kv put networking/configuration/globo-primary/common_tags @common-tags.json
        ```
    3. Write the **configuration** data for additional **workspaces** config:
        ```
        consul kv put networking/configuration/globo-primary/development/net_info @dev-net.json
        consul kv put networking/configuration/globo-primary/qa/net_info @qa-net.json
        consul kv put networking/configuration/globo-primary/production/net_info @prod-net.json
        ```
    4. Create a **development** workspace (can be repeated for **qa** or **production** environment):
        ```
        cd ../../networking
        terraform init -backend-config="path=networking/state/globo-primary" -backend-config="address=127.0.0.1:8500"
        terraform workspace new development
        terraform plan -out dev.tfplan
        terraform apply "dev.tfplan"
        ```
19. Add the **applications** stack to **consul**:
    1. Change to **Sally's** token for **Consul**: `export CONSUL_HTTP_TOKEN=SECRETID_VALUE`
    2. Write the **configuration** data for additional **workspaces** config:
        ```
        cd ../consul/data_configs
        consul kv put applications/configuration/globo-primary/development/app_info @dev-app.json
        consul kv put applications/configuration/globo-primary/qa/app_info @qa-app.json
        consul kv put applications/configuration/globo-primary/production/app_info @prod-app.json
        consul kv put applications/configuration/globo-primary/common_tags @app-tags.json
        ```
    3. Create a **development** workspace (can be repeated for **qa** or **production** environment):
        ```
        cd ../../applications
        terraform init -backend-config="path=applications/state/globo-primary" -backend-config="address=127.0.0.1:8500"
        terraform workspace new development
        terraform plan -out dev.tfplan
        terraform apply "dev.tfplan"
        ```
    4. Open web app page:
       1. Go to **AWS** management console:
        ```
        https://us-east-1.console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances
        ```
       3. Open one of the available instances
       4. Find and copy **public IPv4 Address**
       5. Paste the link into the browser tab
       6. Change **https://** to **http://** (you should be redirected to **wordpress admin** menu)
20. Destroy **stacks**:
    * For **base environment** (repo root):
        ```
        cd ../
        terraform destroy -auto-approve
        ```
    * For **applications**:
        ```
        cd applications
        terraform destroy -auto-approve
        # terraform workspace select development && terraform destroy -auto-approve
        # terraform workspace select qa && terraform destroy -auto-approve
        # terraform workspace select production && terraform destroy -auto-approve
        ```
    * For **networking**:
        ```
        cd ../networking
        terraform destroy -auto-approve
        # terraform workspace select development && terraform destroy -auto-approve
        # terraform workspace select qa && terraform destroy -auto-approve
        # terraform workspace select production && terraform destroy -auto-approve
        ```


### Jenkins configuration ###
1. Make sure that **Consul** is still up-and-running
2. Export **root** token:
    ```
    export CONSUL_HTTP_TOKEN=SECRETID_VALUE
    ```
4. Create two tokens for **Jenkins** (**networking** and **applications** and make note of `SecretID` for later):
    ```
    consul acl token create -policy-name=networking -description="Jenkins networking"
    consul acl token create -policy-name=applications -description="Jenkins applications"
    ```
5. Create a **Jenkins** container (copy the **admin** password):
    ```
    sudo sysctl -w net.ipv4.conf.docker0.route_localnet=1 # enable route_localnet for docker0 interface
    sudo iptables -t nat -A PREROUTING -p tcp -i docker0 --dport 8500 -j DNAT --to-destination 127.0.0.1:8500 # add routing rule for docker0 interface
    docker pull jenkins/jenkins:lts
    docker run --add-host=host.docker.internal:host-gateway -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home --name jenkins jenkins/jenkins:lts
    docker logs jenkins
    ```
6. Go to the http://127.0.0.1:8080 use the **admin** password for logging:
    1. **Terraform** configuration:
        ```
        Install suggested plugins
        Create a user
        Manage jenkins
        Manage plugins
        Search for Terraform in Available and install without restart
        Back to Manage jenkins
        Global Tool Configuration
        Add Terraform
        Name: terraform
        Install automatically
        Version - latest for linux (amd64)
        Click Save
        ```
    2. Create **credentials**:
        ```
        Go to credentials -> global
        Create a credential of type secret text with ID networking_consul_token and the consul token as the secret
        Create a credential of type secret text with ID applications_consul_token and the consul token as the secret
        Create a credential of type secret text with ID aws_access_key and the access key as the secret
        Create a credential of type secret text with ID aws_secret_access_key and the access secret as the secret
        ```
    3. Create pipeline for **networking** stack:
        ```
        Create a new item
        Name: net-deploy
        Type pipeline
        Select poll SCM
        Definition: Pipeline script from SCM
        SCM: Git
        Repo URL: https://github.com/ArseniD/terraform_jenkins_consul.git
        Script path: networking/Jenkinsfile
        Click Save
        Now we can run a build of the network pipeline
        First build might fail, but now the parameters will be Available
        Run a new build WITH parameters
        ```
    4. Create pipeline for **application** stack:
        ```
        Create a new item
        Name: app-deploy
        Type pipeline
        Select poll SCM
        Definition: Pipeline script from SCM
        SCM: Git
        Repo URL: https://github.com/ArseniD/terraform_jenkins_consul.git
        Script path: applications/Jenkinsfile
        Click Save
        Now we can run a build of the application pipeline
        First build might fail, but now the parameters will be available
        Run a new build WITH parameters
        ```
    5. Open web app page:
       1. Go to **AWS** management console:
        ```
        https://us-east-1.console.aws.amazon.com/ec2/v2/home?region=us-east-1#Instances
        ```
       3. Open one of the available instances
       4. Find and copy **public IPv4 Address**
       5. Paste the link into the browser tab
       6. Change **https://** to **http://** (you should be redirected to **wordpress admin** menu)
