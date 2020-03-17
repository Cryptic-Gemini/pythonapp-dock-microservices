# PIAIC-Faisalabad-Kubernetes-Assignment-01
> There are no secrets to success. It is the result of efforts, hard work, and learning from failure.
## Assignment Requirement:

 1. Dockerize python flask application
  
  ## Assignment Help
  
 1. Download/clone the repo
2.  Application require python installations in your system
3.  Go to folder open terminal and  run 'pip install -r requirements.txt' for packages installations. 
4.  You can run application using 'python  app.py' 
5.  Application port 2020. 
6.  You can use application at localhost:2020
 
 Happy Learning! Best of Luck.
------------------------------------------------------------------------------------------------------------------------------------------------

To run your application locally :

step1 : Install python 3.7
(ref: https://linuxize.com/post/how-to-install-python-3-7-on-ubuntu-18-04/)

step2 : Install pip3
(ref: https://linuxize.com/post/how-to-install-pip-on-ubuntu-18.04/)


step3 : Make a virtual environment (first make a directory i.e python-assignment then move into that directory and create virtual env)
(ref1: https://www.geeksforgeeks.org/python-virtual-environment/)(ref2:https://www.youtube.com/watch?v=Lah7WGW6exg)
   
=>command (install virtual env) : pip3 install virtualenv
This doesn't create any virtual environment. That installs virtualenv program, that is used in order to create virtual environments.

What is the default python version that your virtual environment will have is specified as an argument, when actually making the env, like:
=>command (to make virtual env) : virtualenv -p python3 my_venv

step4 : Clone your project in the directory(i.e python-assignment) having virtual env.
(ref: [https://github.com/naveed-rana/PIAIC-Faisalabad-Kubernetes-Assignment-01/])

step5 : Now after creating virtual environment, you need to activate it. Remember to activate the relevant virtual environment every time you work on the project. This can be done using the following command:

=> command : $ source virtualenv_name/bin/activate

step6: run 'pip install -r requirements.txt' for packages installations. 

(missing packages skbuild (pip install scikit-build)and cmake (sudo apt-get install cmake) => again do the step6 

step7 :  You can run application using 'python  app.py' 

step8 : You can use application at localhost:2020

------------------------------------------------------------------------------------------------------------------------------------------------

To Dockerize your flask app using nginx and uwsgi

(ref: https://medium.com/@gabimelo/developing-a-flask-api-in-a-docker-container-with-uwsgi-and-nginx-e089e43ed90e)

==>step1: create your flask app or clone it in a directory (i.e python-assignment)

==>step2: we create an app.ini file so that uWSGI knows how to operate.

; app.ini
[uwsgi]
protocol = uwsgi
; This is the name of our Python file
; minus the file extension
module = app
; This is the name of the variable
; in our script that will be called
callable = app
master = true
; Set uWSGI to start up 5 workers
processes = 5
; We use the port 5000 which we will
; then expose on our Dockerfile
socket = 0.0.0.0:5000
vacuum = true
die-on-term = true

==>step3: Now that our Flask/uWSGI app is set up, we can Dockerize it.We will write a Dockerfile for it. This one, however, will be more minimalistic, since we won’t be installing Nginx in it, and rather running it separately through docker-compose. The file will be named Dockerfile-flask since we will have two Dockerfiles in this project.

# Dockerfile-flask

FROM python:3.6-slim-stretch
RUN apt-get -y update
RUN apt-get install -y --fix-missing \
    build-essential \
    cmake \
    gfortran \
    git \
    wget \
    curl \
    graphicsmagick \
    libgraphicsmagick1-dev \
    libatlas-base-dev \
    libavcodec-dev \
    libavformat-dev \
    libgtk2.0-dev \
    libjpeg-dev \
    liblapack-dev \
    libswscale-dev \
    pkg-config \
    python3-dev \
    python3-numpy \
    software-properties-common \
    zip \
    && apt-get clean && rm -rf /tmp/* /var/tmp/*

RUN cd ~ && \
    mkdir -p dlib && \
    git clone -b 'v19.9' --single-branch https://github.com/davisking/dlib.git dlib/ && \
    cd  dlib/ && \
    python3 setup.py install --yes USE_AVX_INSTRUCTIONS

RUN apt-get update

RUN pip3 install uwsgi

LABEL maintainer="gzlkhan409@gmail.com"

LABEL roll_no="PIAIC61295"

ENV CREATEDBY="Cryptic Gemini"

COPY . /app

WORKDIR /app

RUN  pip3 install flask &&  pip3 install face_recognition

#copy the requirements file to install python dependencies

COPY requirements.txt .

#install python dependencies

RUN pip install -r requirements.txt

#expose the port uwsgi will listen on 

EXPOSE 2020

#finally we run uwsgi with ini file

CMD [ "uwsgi", "--ini", "app.ini" ]


TIP:
You might have found it odd that I first copy requirements.txt into the image and later the rest of the codebase. I do this because Docker creates layers (or intermediate images) as it builds your image. Each layer is cached, and when a file that previously got copied into the image changes, it invalidates its cache and that of all the following layers. Therefore, we can copy a file that barely ever changes first (i.e. requirements.txt) and install modules in one go, before even introducing the rest of the codebase which will most likely change after each build, triggering a re-install of all of our modules/libraries.

==>Step4: Configuring our Generic Nginx Container
Here we will create our configuration file that will tell Nginx how to route traffic to uWSGI in our other container. Our app.conf will essentially replace the /etc/nginx/conf.d/default.conf that the Nginx container includes implicitly.

# app.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    location / { try_files $uri @app; }
    location @app {
        include uwsgi_params;
        uwsgi_pass flask:2020;
    }
}


Note:The line uwsgi_pass flask:2020; is using flask as the host to route traffic to. This is because we will configure docker-compose to connect our Flask and Nginx containers through the flask hostname.


Step5: Building our Nginx Docker Image
Our Dockerfile for Nginx is simply going to inherit the latest Nginx image from the Docker registry, remove the default configuration file, and add the configuration file we just created during build. We won’t even use a CMD instruction, since it will just pick up the one from nginx:latest.
We will name the file Dockerfile-nginx.


# Dockerfile-nginx
FROM nginx:latest
#Nginx will listen on 
EXPOSE 80
#remove the default config file that nginx.conf includes
RUN rm /etc/nginx/conf.d/default.conf
#we copy the app.conf to conf.d
COPY app.conf /etc/nginx/conf.d

TIP#2:

You will notice later on this article that I am exposing the port for this container both in its Dockerfile and on docker-compose.yml. I do this because commonly people don’t use docker-compose in production. I use it in development and then I deploy my containers through some separate service (e.g. Kubernetes, ECS, or Heroku). So by exposing the ports in both places, I ensure that different services will know which port to route connections to by just looking at the Dockerfile.

Step6: Orchestrating with Docker Compose
Now that our configuration is ready to run the Flask/uWSGI container, we can write the necessary configuration so that the entire stack will run just by typing docker-compose up in our terminal.
All the configuration for docker-compose goes in a YML file called docker-compose.yml. This file usually lives in the root of your project so that running any compose command will automatically pick it up.

# docker-compose.yml
version: '3'
services:
  flask:
    image: webapp-flask
    build:
      context: .
      dockerfile: Dockerfile-flask
    volumes:
      - "./:/app"
  nginx:
    image: webapp-nginx
    build:
      context: .
      dockerfile: Dockerfile-nginx
    ports:
      - 5000:80
    depends_on:
      - flask



step7: Now you can run docker-compose up to run your entire app instead of having to run all these commands:
$ docker build -t my-flask -f Dockerfile-flask .
$ docker build -t my-nginx -f Dockerfile-nginx .
$ docker network create my-network
$ docker run -d --name flask --net my-network -v "./:/app" my-flask
$ docker run -d --name nginx --net my-network -p "5000:80" my-nginx

step8: Now that your Docker containers are running, feel free to rush to localhost:2020 to see your new web app live running in two separate containers!

------------------------------------------------------------------------------------------------------------------------------------------------
Dissection time!
Let’s walk through the important lines on our docker-compose.yml file to fully explain what is going on.
The keys under services: define the names of each one of our services (i.e. Docker containers). Hence, flask and nginx are the names of our two containers.

==> image: webapp-flask

This line specifies what name our image will have after docker-compose creates it. It can be anything we want. docker-compose will build the image the first time we launch docker-compose up and keep track of the name for all future launches.

==>build:
  context: .
  dockerfile: Dockerfile-flask

This piece is doing two things. First (i.e. context), it is telling the Docker engine to only use files in the current directory to build the image. Second (i.e. dockerfile), it’s telling the engine to look for the Dockerfile named Dockerfile-flask to know the instructions to build the appropriate image.

==>volumes:
  - "./:/app"

Here we’re simply instructing docker-compose to mount our current folder onto the directory /app in the container when it is spun up. This way, as we make changes on the app, we won’t have to keep building the image unless it is a major change, such as a software module dependency.
For the nginx portion of the file, there’s a few things to look out for.

==>ports:
  - 2020:80

This little section is telling docker-compose to map the port 2020 on your local machine to the port 80 on the Nginx container (which is the port Nginx serves to by default). This is why going to localhost:2020 is able to hit your container.


==>depends_on:
  - flask

This part is critical. As you might have noticed in the app.conf, we route traffic from Nginx to uWSGI and viceversa by sending data through the flask hostname. What this section does, is create a virtual hostname flask in our nginx container and setup the networking so that we can route the incoming data to our uWSGI app living in a different container. The depends_on directive also waits until the flask container is in a functional state before launching the nginx container, which avoids having a scenario where Nginx fails when the flask host is unresponsive.

-----------------------------------------------------------------------------------------------------------------------------------------------



















