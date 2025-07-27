**Since we successfully configured Docker on our WSL2 Linux VM using Ansible in Part 2, the next step was to use Docker Compose to run the todo-app alongside a MongoDB container, and to choose an appropriate tool to enable automatic updates

**Created Docker Compose Deployment Directory
On the WSL2 VM, I created a new directory to hold deployment-related configuration files:

mkdir ~/todo-deployment
cd ~/todo-deployment

**Had to choose the auto update tool:

I chose Watchtower as my auto-update tool, as it-
1. meets the exact requirement (monitors the running containers, checks the registry for newer images, pulls updates, and replaces the containers seamlessly. No scripting, no manual polling, no CI/CD platform needed)
2. Needs zero setup overhead
3.Fully Docker-Native
4.Real-time polling with fine control with only one line (command: --interval 30 todo-app)
5.Supports private registries. (I had to change my registry from public to private so you can evaluate and assess my work :)

**Created docker compose file:

nano docker-compose.yml

**Content:

services:
  mongo:
    image: mongo
    container_name: mongo
    restart: always
    ports:
      - "27018:27017"  # Chose 27018 because 27017 was already in use on local host
    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - mongo_data:/data/db

  todo-app:
    image: omaraboagla/fortstakassessment:latest
    container_name: todo-app
    restart: always
    ports:
      - "4000:4000"
    depends_on:
      mongo:
        condition: service_healthy
    environment:
      - MONGO_URI=mongodb://mongo:27017/todoapp

  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --interval 30 todo-app

volumes:
  mongo_data:

**In my docker compose file, I had to change Mongo's host port to 27018 as to avoid conflict with the container already running from Part 1:

**Added healthchecks in my Docker Compose setup to continuously verify the actual working state of each service, not just its process status:

    healthcheck:
      test: ["CMD", "mongo", "--eval", "db.adminCommand('ping')"]
      interval: 10s
      timeout: 5s
      retries: 5

**Healthcheks implemented: 

Runs a ping command via the MongoDB shell to verify the database is responsive through an interval that runs the test every 10 seconds, timeout set so that it fails the check if it doesn’t respond within 5 seconds; and only marks the container as "unhealthy" after 5 consecutive failures.

**Brought services up:

docker compose up -d

**All services started successfully after fixing the port conflict. The Mongo container became healthy as verified using:

docker inspect mongo | grep -A10 Health

**Verified Services Were Running:

docker ps

**Confirmed the following were running:

mongo on 27018
todo-app on 4000
watchtower (monitoring updates)

**Now comes the fun part :), testing Watchtower Auto-Update:

Locally (on Windows), I changed a visible element inside the app (an <h1> that said "Todos, just tasks" in dashboard.ejs to be "Todos, just tasks This is a change test"):


**Rebuilt and pushed the Docker image:

docker build -t omaraboagla/fortstakassessment:latest .
docker push omaraboagla/fortstakassessment:latest

**Waited 30–60 seconds, then checked Watchtower logs:

docker logs watchtower

**Watchtower output showed it found a new image and restarted todo-app, it returned the following, which clearly indicates it found new image, stops the current onem creates and updates the new one!!

time="2025-07-26T20:04:02Z" level=info msg="Found new omaraboagla/fortstakassessment:latest image (1c3398e52f73)"
time="2025-07-26T20:04:02Z" level=info msg="Stopping /todo-app (7697a53fb8bd) with SIGTERM"
time="2025-07-26T20:04:03Z" level=info msg="Creating /todo-app"
time="2025-07-26T20:04:04Z" level=info msg="Session done" Failed=0 Scanned=1 Updated=1 notify=no
time="2025-07-26T20:04:15Z" level=info msg="Session done" Failed=0 Scanned=1 Updated=0 notify=no

**refreshed my page "Localhost:4000" and it shows the header:

Todos, just tasks This is a change test