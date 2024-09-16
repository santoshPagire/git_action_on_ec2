# Deploy simple node.js application on EC2 using Github Action

Steps:
## 1. Create node js application using following commands
- update packages and libraries
```bash
sudo apt update
```
- install npm
```bash
npm init -y
```
- install express
```bash
npm install express
```
![alt text](<images/Screenshot from 2024-09-16 10-12-00.png>)
- create 'server.js' file and add followng content in it.
```js
const express = require('express')

const app = express()



app.get('/', (req, res) => {
    res.json({
        message: "Hello World"
    })
})

app.listen(8080, () => {
    console.log('server running on port 8080')
})
``` 
- run app
```bash
node server.js
```
- check app is running fine
```bash
http://localhost:8080
```
![alt text](<images/Screenshot from 2024-09-16 14-55-42.png>)
## 2.Create Dockerfile to build image 
- Dockerfile
```yml
FROM node:16

WORKDIR /usr/src/app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 8080
CMD [ "node", "server.js" ]
```
## 3.Create git repository and push code to repo
```bash
git init
git add .
git commit -m "code added"
git branch -M main
git remote add origin git@github.com:santoshPagire/git-test-commands.git
git push -u origin main
```
## 4.Create '.github/Workflows' directory and inside it create build. yml file and add following content in it.
- build.yml

```yml
name: Build Pipeline

on:
  push:
    branches:
      - main

jobs:

  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Login Dockerhub
      
      env:
        DOCKER_USERNAME: ${{secrets.DOCKER_USERNAME}}
        DOCKER_PASSWORD: ${{secrets.DOCKER_PASSWORD}}
      run: docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
      
    - name: Build the Docker image
      run: docker build -t santoshpagire/git-action-node-app .

    - name: Push to Dockerhub
      run: docker push santoshpagire/git-action-node-app
```
## 5.Add credentials in repository secrets
- Go to repository settings.
- On left side menu click on "Secrets and Variables" then click on "Actions"
- Click on "New repository secret" and add secrets.(i.e. DOCKER_USERNAME and DOCKER_PASSWORD )
![alt text](<images/Screenshot from 2024-09-16 14-05-32.png>)

Now push the build.yml to repository and it will trigger pipeline.

To check pipeline go to your repository and on upper side click on "Actions" you will the pipeline.

This workflow will create docker image and push it to dockerhub.
![alt text](<images/Screenshot from 2024-09-16 14-49-03.png>)
![alt text](<images/Screenshot from 2024-09-16 14-56-02.png>)

## 6.Add self hosted runner (EC2)
- Click on repository "Settings"
- Click on "Actions" on left hand side then click on "Runners".
- Click on "New self-hosted runner" then choose OS 'ubuntu' and then it will show couple of commands to run on your Ec2 to connect it with github.
- Create Ec2 instance on Aws and configure security group for ssh,http and required ports (i.e 8080).
![alt text](<images/Screenshot from 2024-09-16 14-43-49.png>)
![alt text](<images/Screenshot from 2024-09-16 14-44-37.png>)
- SSh into EC2 and run the commands given by github to connect runner. 
- Commands for reference
```bash
# Create a folder
$ mkdir actions-runner && cd actions-runner
# Download the latest runner package
$ curl -o actions-runner-linux-x64-2.319.1.tar.gz -L https://github.com/actions/runner/releases/download/v2.319.1/actions-runner-linux-x64-2.319.1.tar.gz
# Optional: Validate the hash
$ echo "3f6efb7488a183e291fc2c62876e14c9ee732864173734facc85a1bfb1744464  actions-runner-linux-x64-2.319.1.tar.gz" | shasum -a 256 -c
# Extract the installer
$ tar xzf ./actions-runner-linux-x64-2.319.1.tar.gz

# Configure

# Create the runner and start the configuration experience
$ ./config.sh --url https://github.com/santoshPagire/git_action_on_ec2 --token A6YNPUDZHEZLKD6OGK3DNJ3G47WRW
# Last step, run it!
$ ./run.sh

```
![alt text](<images/Screenshot from 2024-09-16 14-01-45.png>)
![alt text](<images/Screenshot from 2024-09-16 14-02-16.png>)
![alt text](<images/Screenshot from 2024-09-16 14-03-02.png>)
![alt text](<images/Screenshot from 2024-09-16 14-03-37.png>)

- Now install Docker on EC2 
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
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
![alt text](<images/Screenshot from 2024-09-16 14-00-53.png>)
![alt text](<images/Screenshot from 2024-09-16 14-01-09.png>)
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
![alt text](<images/Screenshot from 2024-09-16 14-01-19.png>)
- Give sudo permissions 
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```
![alt text](<images/Screenshot from 2024-09-16 14-02-36.png>)

## 7.Create deploy.yml file under '.github/Workflows' directory and add following code.
- deploy.yml
```yml
name: Deploy Pipeline

on:
  workflow_run:
    workflows: ["Build Pipeline"]
    types:
      - completed

jobs:

  build:

    runs-on: self-hosted 

    steps:
     - name: Login Dockerhub
         
       env:
         DOCKER_USERNAME: ${{secrets.DOCKER_USERNAME}}
         DOCKER_PASSWORD: ${{secrets.DOCKER_PASSWORD}}
       run: docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD

     - name: Pull Docker image
       run: sudo docker pull santoshpagire/git-action-node-app:latest

     - name: Delete Old docker container
       run: sudo docker rm -f node-app-container || true
       
     - name: Run Docker Container
       run: sudo docker run -d -p 8080:8080 --name node-app-container santoshpagire/git-action-node-app
```
Now push the deploy.yml file to repository and it will trigger build pipeline and after that deploy pipeline will trigger as it depends on build pipeline.

This workflow will pull docker image and run container on EC2 runner.
![alt text](<images/Screenshot from 2024-09-16 14-52-37.png>)

## 8. Verify output
- Check docker container successfully run on EC2
```bash
docker ps
```
![alt text](<images/Screenshot from 2024-09-16 14-24-11.png>)
- Check output on browser
```bash
http://public-ip-of-ec2:8080
```
![alt text](<images/Screenshot from 2024-09-16 13-58-03.png>)
- Make some changes  in server.js file and push it to repo and check everything working fine and also varify changed message.
![alt text](<images/Screenshot from 2024-09-16 14-04-51.png>)
![alt text](<images/Screenshot from 2024-09-16 14-24-27.png>)




