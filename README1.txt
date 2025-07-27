**cloned repository using:
git clone https://github.com/Ankit6098/Todo-List-nodejs
cd Todo-List-nodejs

**At:
C:\Users\Legion\Todo-List-nodej

**Ran MongoDB locally in Docker:
docker run -d -p 27017:27017 --name mongo mongo

**Updated .env to use my own MongoDB:
MONGO_URI=mongodb://host.docker.internal:27017/todoapp

**Created Dockerfile:
FROM node:18

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]

**built image using:
docker build -t todo-app .

**Faced errors due to .env file returning Null, inspected index.js to load the .env file and use process.env.MONGO_URI properly I added:
require('dotenv').config() in index.js
Without that line, process.env.MONGO_URI will be undefined, and Mongoose will crash trying to connect

**Error was still persistent, Even though I added require('dotenv').config() in index.js, it might be needed in config/mongoose.js too if mongoose.js is being executed before index.js hence amended mongoos.js:

**At the top of /config/mongoose.js, added:

Copy code
require('dotenv').config();
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
});

**In mongoose.js, right after require('dotenv').config();, added:

console.log("MONGO_URI from ENV:", process.env.MONGO_URI);

**Then rebuilt and re-ran the container:

docker build -t todo-app .
docker run -p 4080:4000 --env-file .env todo-app

**Received "Yupp! Server is running on port 4000", tested localhost:4080 and angular app loaded successfully
**Tagged the image using:
docker tag todo-app omaraboagla/todo-app:latest

**Pushed the image using:
docker push omaraboagla/todo-app:latest

**Created a GitHub Repo to store the project and set up GitHub Actions:
---
**repo name: omaraboagla/fortstakassessment

**Linked Local Project to GitHub, project had an origin that I needed to remove before adding my origin:

git init (Initializes Git tracking.)
git remote remove origin (Links my local folder to the GitHub repo)
git remote add origin https://github.com/YOUR_USERNAME/todo-app-docker.git
git branch -M main (Renames current branch to main.)
git add .
git commit -m "Initial commit"
git push -u origin main (Uploads project files to GitHub)

**Created GitHub Secrets:
**added two repository secrets in GitHub:
DOCKER_USERNAME
DOCKER_PASSWORD

**Created GitHub Actions Workflow File:
Created .github/workflows/docker-publish.yml

**docker-publish.yml Contains:
name: Build and Push Docker Image

on:
  push:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}


      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: omaraboagla/fortstakassessment:latest

**Commit & Push:
git add .
git commit -m "Add GitHub Actions to build & push image"
git push

***Error Encountered "failed to build: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory", had to explicitly tell GitHub Actions where to find the Dockerfile (even though it was in the root, the build system couldn’t locate it automatically) by amending the .yml file adding:

with:
       context: .
       file: ./Dockerfile

**Commit & Push:
git add .
git commit -m "Fix Dockerfile path for Github Actions"
git push

**GitHub Actions now:
1.Detects new code pushes to main
2.Builds my Docker image from the Dockerfile
3.Logs into Docker Hub
4.Pushes the image to the repo:
➜ omaraboagla/fortstakassessment
Verified it appeared in Docker Hub, and everything works!