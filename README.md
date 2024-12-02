# CI/CD with Gitea and ArgoCD on Internet-free Environment

## Assumptions
> We assume that your host machine has internet access and your servers haven't and you already created a rke2 kubernetes cluster including your server machines.
Also you use your private registry to pull images or install things.(All machines use Rocky9 as OS.)

## 1. ArgoCD Deployment on K8s

### Configuration For Private Registry
> Before deploying ArgoCD you have to configure  /var/lib/rancher/rke2/agent/etc/containerd/certs.d/_default/hosts.toml  to use your private registry.

/var/lib/rancher/rke2/agent/etc/containerd/certs.d/_default/hosts.toml :   

```
# File generated by rke2. DO NOT EDIT.

server = ""
capabilities = ["pull", "resolve", "push"]


[host]
[host."<your-private-registry>:<port1>"]
  capabilities = ["pull", "resolve"]
[host."<your-private-registry>:<port2>"]
  capabilities = ["pull", "resolve"]


```
By adding your private registry to this file you will be able to make deployments without configuring images to use your private registry one-by-one.

### Adding Install YAML

Download the install.yaml from the following link on your host machine and send it to your master server:
https://github.com/argoproj/argo-cd/blob/master/manifests/install.yaml

### Creating ArgoCD Namespace and Applying Deployment

```bash
kubectl create ns argocd
```
This command creates argocd namespace.  
<br>
```bash
kubectl apply -n argocd -f install.yaml
```
Apply argocd deployment.  
<br>

```bash
kubectl get pods -n argocd
```
Make sure that each pod works successfully.

### ArgoCD UI Login
####  Option 1 - Port Forwarding

Run the following command to forward argocd-server's 443 port to server machine's 8080 port:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
If you get "port already in use" error you can try 8000 instead of 8080.

Don't end this process while you're accessing to UI.  
<br>

Then on your host PC, open powershell and run the following command:
```bash
ssh -L 8080:localhost:8080 user@<server-machine-ip>
```
The server-machine-ip is your master node's IP since you run the port-forward command there.  

<br>

After this you can acces your server machine's 8080 port from your host PC's 8080 port. Go to the web browser on your host PC and type following:
```txt
http://localhost:8080
```

Now, you have to be able to view Argocd UI.

Default username:
```txt
admin
```
Password:  
Run the following command to get your password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```


#### Option 2 - Node Port
You can open your node to an external access by using NodePort services. To enable it,

```bash
kubectl expose deployment argocd-server --name=argocd-server-np --port=80 --target-port=8080 --type=NodePort --namespace=argocd 
```
The command above exposes the argocd-server to a NodePort service named argocd-server-np.


You can check which port the argocd-server-np service uses by running the following command. (It's in >=3000 range)
```bash
kubectl get svc -n argocd
```
Keep in mind the port number, we will use it.

<br>

Run the following command to learn the node where argocd works:
```bash
 kubectl get pods -n argocd -o wide
```
Remember the node name.

Run the follwing command and look for the IP of that node.
```bash
kubectl get nodes -o wide
```

Then go to the browser on your host PC and type following:
```txt
http://<node-ip>:<node-port>
```
Now, you have to be able view Argocd UI.

Default username:
```txt
admin
```
Password:  
Run the following command to get your password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode ; echo
```


## 2. Gitea Set-up with Docker

#### Docker Installation 
Make sure you have added your private Docker repository.

```bash
yum install docker-ce
```

Start Docker Engine.
```bash
systemctl enable --now docker
```
 Login Docker.
 ```bash
 docker login <registry.example.com>:<port>
 ```

 <br>

 Create gitea directory:
 ```bash
mkdir gitea
 ```
 Inside gitea directory create a file named docker-compose.yaml:
 ```bash
 cd gitea
 touch docker-compose.yml
 ```

 docker-compose.yml:
 ```txt
 version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: <your-private-registry>:<port>/gitea/gitea:1.22.3
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=mysql
      - GITEA__database__HOST=db:3306
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
    depends_on:
      - db

  db:
    image: <your-private-registry>:<port>/mysql:8
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql
 ```
You can use the yaml above then save and exit.  


Run:
```bash
docker compose up -d
```

```txt
Then go to http://<IP-of-the-node-gitea-placed>:3000 on your browser
```
After you complete initial configurtion you will be able to create your project.

### Creating Gitea Runner
> We will create a Runner to ensure auto-build for our project images and updating our deployment yamls.

Add the followings to the end of the docker-compose.yml:
```txt
runner:
    image: <your-private-registry>:<port>/gitea/act_runner:nightly
    environment:
      GITEA_INSTANCE_URL: "http://<node-IP-gitea-works>:3000"
      GITEA_RUNNER_REGISTRATION_TOKEN: "<token>"
      GITEA_RUNNER_NAME: "Gitea Runner"
      DOCKER_CLI_EXPERIMENTAL: "enabled"
    depends_on:
      - server
      - db
    networks:
      - gitea
    volumes:
      - ./runner:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/root/.docker/config.json

```
```txt
<token> : Open your project on gitea and go Settings. On the left menu, click Actions than click Runners on opened menu. Then on the right side of the window, click Create new Runner and copy the token.

Settings > Actions > Runner > Create new Runner
``` 
Run the following command to save login credentials to /docker/config.json:
```bash
docker login -u <username> -p <password> <your-private-registry>:<port>
```
If you use different repositories for pull and push operations, don't forget to login for both.  
<br>

Your docker-compose.yml should look like this:

```txt
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: <your-private-registry>:<port>/gitea/gitea:1.22.3
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=mysql
      - GITEA__database__HOST=db:3306
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
    depends_on:
      - db

  db:
    image: <your-private-registry>:<port>/mysql:8
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql
  runner:
    image: <your-private-registry>:<port>/gitea/act_runner:nightly
    environment:
      GITEA_INSTANCE_URL: "http://<node-IP-gitea-works>:3000"
      GITEA_RUNNER_REGISTRATION_TOKEN: "<token>"
      GITEA_RUNNER_NAME: "Gitea Runner"
      DOCKER_CLI_EXPERIMENTAL: "enabled"
    depends_on:
      - server
      - db
    networks:
      - gitea
    volumes:
      - ./runner:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/root/.docker/config.json
```

### Creating Build-and-Push Pipeline

Create the folders on your project :
```bash
cd <your-project-folder>
mkdir -p .gitea/workflows
touch .gitea/workflows/pipeline.yml
```
pipeline.yml (base):
```txt
name: Build and Push Docker Images
run-name: ${{ gitea.actor }} is building Docker images 

on:
  push:
    branches:
      - main  # Trigger on push to 'main' branch, modify as needed

jobs:
  Build-and-Push:
    runs-on: ubuntu-latest
    steps:
      # Notify about the event
      - run: |
          echo " Triggered by a ${{ gitea.event_name }} event."
          echo " Branch: ${{ gitea.ref }}, Repository: ${{ gitea.repository }}."                    


      # Step 1: Check out the repository code
      - name: Check out repository code
        uses: actions/checkout@v4

      
      # Step 2: Build and Push Docker Image
      - name: Build and Push Image
        run: |
          docker login  <your-private-registry>:<port> -u <username> -p <password>
          docker build -t <your-private-repo>:<port>/<image-name>:${{ gitea.sha }} ./<Dockerfile-path>
          docker push <your-private-repo>:<port>/<image-name>:${{ gitea.sha }} ./<Dockerfile-path>                
           

      # Step 3: Update image tags in Deployment YAMLs
      - name: Update Image Tags
        run: |
          new_image_tag=${{ gitea.sha }}
          sed -i "s|<your-private-repo>:<port>/<image-name>:[^ ]*|<your-private-repo>:<port>/<image-name>:$new_image_tag|" 
          <path-of-deployment-yaml>
      
      
      # Step 4: Commit Changes
      - name: Commit Changes  
        run: |
          git config --global user.name '<username>'
          git config --global user.email '<password>'
          git commit -am "updating image tag"
          git push origin main          


      # Final status
      - run: |
          echo " Job completed with status: ${{ job.status }}."                
```
```txt
runs-on: This part allows labels. You have to make some configurations to be able to use your private registry.
```
### Label Configuration for runs-on 

Run the following under gitea folder:
```bash
docker run --entrypoint="" --rm -it gitea/act_runner:latest act_runner generate-config > runner-config.yaml
```
Open the runner-config.yaml and find labels part then make the proper changes as the following: 
```txt
labels:
    - "ubuntu-latest:docker://<your-private-registry>:<port>/gitea/runner-images:ubuntu-latest"
```
Open docker-compose.yml and add runner-config.yaml to volumes and create CONFIG_FILE environment variable. The runner part should be look like the following:

```txt
runner:
    image: <your-private-registry>:<port>/gitea/act_runner:nightly
    environment:
      CONFIG_FILE: /config.yaml
      GITEA_INSTANCE_URL: "http://:3000"
      GITEA_RUNNER_REGISTRATION_TOKEN: "gkeYdqVI5ZVLUFselfLrqCi9yrGoyd6a8dAwLGpR"
      GITEA_RUNNER_NAME: "Gitea Runner-4"
      DOCKER_CLI_EXPERIMENTAL: "enabled"
    depends_on:
      - server
      - db
    networks:
      - gitea
    volumes:
      - ./runner-config.yaml:/config.yaml
      - ./runner:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/root/.docker/config.json
```  


Run the following to start runner:
```bash
docker compose up -d
```
Now you can check Actions section in Gitea UI wether pipeline works properly. 

### Generating a Webhook to Sync Immediately
> Even though you choose auto-sync option while creating the app,
changes will be applied in 3-5 minutes in ArgoCD. To ensure changes to be applied immediately we will use webhooks.

```txt
Go Settings in Gitea UI, click on Webhooks on the left-side menu then click the Add Webhook and choose Gitea.

Settings > Webhooks > Add Webhook > Gitea
```
```txt
For the Target URL,

if you used port-forwarding:
Use http://localhost:<the-port-you-used>/api/webhook

if you used NodePort:
Use http://<argocd-server-node-IP>:<NodePort>/api/webhook
```
```txt
If you click refresh button on ArgoCD UI you will be able to see the changes immediately.
```

### Deploying an App with ArgoCD
```txt
Open ArgoCD UI on Applications menu click NEW APP button on the left-up corner.

Applications > NEW APP
```
<br>

```txt
On GENERAL part,
```

```txt
For SYNC POLICY choose Automatic to ensure AUTO-SYNC.

For SYNC OPTIONS, 

- choose PRUNE LAST to prune old resources from the old commit.

- choose AUTO-CREATE NAMESPACE to automatically create a namespace for the project.

For PRUNE PROPAGATION POLICY choose REPLACE to delete all resources 
and create all from scratch on new commit.
```
<br>

```txt
On SOURCE part,
```
```txt
Paste your repository URL.

Write the path of your YAML files in project repo to PATH section.
```
<br>

```txt
Then you can create the app by pressing the CREATE button.
```

## Additional Tips

### 1. Deleting Unused Deployment Replicas

> Even if you have choosed the prune-last option and only the old pods are deleted after the new commit, you can add the following in your deployment YAMLs.

```bash
revisionHistoryLimit: 0
```
>You can add this line under replicas in spec part.
This will delete all the old replicas and allow only the last committed replica remain. You can change the number suported with this specification to adjust the remaining replicas.


### 2. Changing Secret 
> Sometimes argocd raises an issue while trying to login to UI even you give the correct username and password. To solve this problem you can regenerate argocd-secret.

1. Delete the existing argocd-secret by:
```bash
kubectl delete secret argocd-secret
```
2. Generate a new bcrypt-encrypted admin password.
```bash
NEW_PASSWORD=$(htpasswd -nbBC 10 "" <new-password> | tr -d ':\n' | sed 's/^$2y/$2a/')
```
3. Create a secret with new password.
```bash
kubectl create secret generic argocd-secret \
  --from-literal=admin.password="$NEW_PASSWORD" \
  --from-literal=admin.passwordMtime="$(date +%FT%T%Z)"

```
