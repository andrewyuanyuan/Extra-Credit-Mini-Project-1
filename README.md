# Extra-Credit-Mini-Project-1

**Reference**:  [Learn Kubernetes in Under 3 Hours: A Detailed Guide to Orchestrating Containers](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882/)

**This is really an wonderful article, really helps a lot in helping me to have a deeper understanding on Docker and Kubernetes.**

## Steps to get application work

### 1. Running the application on my computer

Before deploying this application on GCP, I tried to run three parts of this application on my local machine to get a basic idea of how different parts interact with each other.

Since all the source codes are provided, we only need to do the following command to build this project.

* Frontend (Running on `localhost:3000`)

  ```bash
  npm install
  npm start
  npm run build
  ```

* Spring Web Application (Running on `localhost:8080`)

  ```bash
  mvn install
  java -jar sentiment-analysis-web-0.0.1-SNAPSHOT.jar 
       --sa.logic.api.url=http://localhost:5000
  ```

  `--sa.logic-api.rul = http://localhost:5000` defines where the Python application is running, it let the Spring Web Application know where to forward messages on run time. In here, we define our python application will run on `localhost:5000`.

* Python application (Running on `localhost:5000`)

  ```bash
  python -m pip install -r requirements.txt
  python -m textblob.download_corpora
  python sentiment_analysis.py
  ```

  The Flask application listen for requests on `0.0.0.0:5000` (calls to `localhost:5000` will reach this app as well)

Now, the first step is done and the application can work on my local machine now.

### 2. Building the container images for three parts of this application and Running the containerized application on local machine

* **Frontend**

  For frontend, we first need to build the static files

  ```bash
  npm run build
  ```

  The we define the `Dockerfile` for SA-Frontend

  ```bash
  FROM nginx
  COPY build /usr/share/nginx/html
  ```

  This Dokerfile means that we start from nginx image, copy the react build directory to `nginx/html` folder in the image.

  Then I pushed it to my docker hub using the following commands:

  ```bash
  docker build -f Dockerfile -t anyuanyu/sentiment-analysis-frontend .
  docker push anyuanyu/sentiment-analysis-frontend
  ```

  To test whether I successfully uploaded it, I pulled the image and run it on my local machine.

  ```bash
  docker pull anyuanyu/sentiment-analysis-frontend
  docker run -d -p 80:80 anyuanyu/sentiment-analysis-frontend
  ```

  Now, if I open `localhost:80` in my browser, the react page will show up.

* **Spring Web Application and Python application**

  For these two parts, I basically do the same thing as frontend except the dockerfiles are different, you can see them in `Sentiment-Analysis` folder.

  The commands I used for them is as followed,

  ```bash
  docker build -f Dockerfile -t anyuanyu/sentiment-analysis-web-app .
  docker build -f Dockerfile -t anyuanyu/sentiment-analysis-logic .
  ```

* **Running the containerized application**

  ```bash
  docker run -d -p 5050:5000 anyuanyu/sentiment-analysis-logic
  docker run -d -p 8080:8080 -e SA_LOGIC_API_URL= 172.17.0.2:5000 anyuanyu/sentiment-analysis-web-app
  docker run -d -p 80:80 anyuanyu/sentiment-analysis-frontend
  ```

### 3. Deploy docker images to GCP

In this step, I used the following commands to deploy these three images I created to the **container registry** of Google cloud platform.

**For frontend:**

```bash
docker pull anyuanyu/sentiment-analysis-frontend:deployment

docker tag anyuanyu/sentiment-analysis-frontend:deployment gcr.io/the-byway-327123/anyuanyu/sentiment-analysis-frontend:deployment

docker push gcr.io/the-byway-327123/anyuanyu/sentiment-analysis-frontend:deployment
```

**For Web-app and logic part:**

It's almost the same, but only replace the docker image name.

**Now, you can see all three images are in my container registry of GCP.**

![screenshot1]()

### 4. Get the application work on Google Kubernetes Engine

To deploy our application on Google Kubernetes Engine, 

**First, I created my kubernetes cluster**, the command I used is the same as the script mention in class. By doing this, I now have this kubernetes cluster which is shown as below:

![screenshot2]()

After creating my clusters, I think we finally face **the most tricky part** of this project. **In the following step, all the files for deployment and creating service is under `/Sentiment-Analysis/resource-manifests` folder.**

For this application, **we must strictly follow a specific deploying order**, which is **logic → web app→frontend**, if we switch this order, the whole deployment will fail and we need to do some parts of the deployment again.

**The reason we have to follow this order is that the logic part of this application depends on web-app part, and only after we have the external IP for web-app part, can we know where the HTTP request in frontend should post.**

1. Therefore, first I deployed the logic part of this application using the following commands

   ```bash
   //I used the raw file provided by Github for all the deployment files and service-create files
   kubectl apply -f sa-logic-deployment.yaml --record 
   kubectl apply -f service-sa-logic.yaml
   ```

   By doing this, I created `sa-logic` service whose type is `Cluster IP`, which means we it's only visible inside clusters and don't have an external IP for it.

2. Then I deployed and created the `sa-web-app-lb` service, which together with `sa-logic`, will serve as the backend for our application, the commands I used are as followed

   ```bash
   kubectl apply -f sa-web-app-deployment.yaml --record
   kubectl apply -f service-sa-web-app-lb.yaml
   ```

   After that, we now finally have the endpoints address for our application backend, in my case, this address is `34.67.118.55:80`, we now need to go back to our frontend code and change the scripts below to provide a valid request address for our frontend

   ![screenshot3]()

 3. Now we need to rebuild our frontend and update our docker image and do the `pull, tag, push`  things again in GCP before we deploy and create service. After that we can finally use these command to deploy the frontend of our application.

    ```bash
    kubectl apply -f sa-logic-deployment.yaml --record
    kubectl apply -f service-sa-logic.yaml
    ```

### 5. Conclusion

After all the steps above, Now we have three services in my Kubernetes Engine of GCP, which is shown in the picture below

![screenshot4]()

The application also successfully running on the external IP through my Kubernetes Service, you can see the IP address is the same as what shown in services.

![screenshot5]()

![screenshot6]()

## Screenshots

Please see the folder named `Screenshots`, the pictures in there are also displayed above in section 1.

## URLs for all my Docker Hub images

[anyuanyu/sentiment-analysis-frontend](https://hub.docker.com/repository/docker/anyuanyu/sentiment-analysis-frontend)

[anyuanyu/sentiment-analysis-logic](https://hub.docker.com/repository/docker/anyuanyu/sentiment-analysis-logic)

[anyuanyu/sentiment-analysis-web-app](https://hub.docker.com/repository/docker/anyuanyu/sentiment-analysis-web-app)

