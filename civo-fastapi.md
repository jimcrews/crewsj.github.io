Let's create a FastAPI app, dockerize it, and deploy it to Civo. Although, before jumping in, I got stumped on the following issue that relates to running container images on different architecture. After I created the FastAPI container image, I tested it locally using k3d, then deployed it to CIVO cloud. However, the Pods would not start, and the Deployment was in error **MinimumReplicasUnavailable**. The Pods were logging error: `exec /usr/local/bin/uvicorn: exec format error`.

The problem is that the image was built on my mac which has an M1 chip (ARM-based), so by default the Docker build command targets `arm64`. In Civo, the the Nodes *Architecture* is listed as `amd64`. 

To resolve, rebuild Docker image, targeting amd64: `--platform=linux/amd64`

## FastAPI
Create a simple FastAPI API using the Docs - [https://fastapi.tiangolo.com](https://fastapi.tiangolo.com/#create-it) 

1. create virtual python env: ```python -m venv .venv```
2. activate env: ```source .venv/bin/activate```
3. pip install fastapi ```pip install fastapi```
4. pip install uvicorn ```pip install uvicorn```
5. create simple app in main.py
6. run app locally: ```uvicorn main:app --reload```
7. test app locally: ```http://127.0.0.1:8000/```
8. stop local uvicorn

## Docker
Dockerize the app using the Docs - [https://fastapi.tiangolo.com/deployment/docker/](https://fastapi.tiangolo.com/deployment/docker/) 

1. create Dockerfile (https://fastapi.tiangolo.com/deployment/docker/)
2. build docker image with tag: ```docker build -t k8s-fast-api .```
3. run container forwarding port: ```docker run -p 8000:80 k8s-fast-api```
4. test app locally: ```http://127.0.0.1:8000/```
5. create Dockerhub repo ([https://hub.docker.com/](https://hub.docker.com/))
6. build and tag with new Dockerhub tag ```docker build --platform=linux/amd64 -t jimcr/fast-api-k8s:0.0.2 .```
7. login to docker using terminal: ```docker login```
8. push to Dockerhub: ```docker push jimcr/fast-api-k8s:0.0.1```


## Civo
Signup to Civo to create a Kubernetes Cluster - [https://civo.com/](https://dashboard.civo.com/)

1. Create a new cluster using defaults
2. Download kubeconfig file and use it to connect to Civo ```export KUBECONFIG=/Users/jimcrews/repos/tiny_k8s_projects/fastapi/kubeconfig```
3. Test kubectl connection by listing the Cluster nodes: ```k get nodes```

## Kubernetes
Create the required Kubernetes resources.

1. Create a basic Deployment, using the *fast-api-k8s* image.
2. Create a basic Service, pointing to the Deployment.
3. Create a basic Traefik Ingress ([https://doc.traefik.io/traefik/providers/kubernetes-ingress/](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)) pointing to the Service.

The Ingress will need to specify what fully qualified hostname it is servicing. Grab the external IP address from the Civo Cluster information page:
![Civo Cluster Information](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/civo-cluster-information.png)

Create a DNS A record:
![Civo DNS Record](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/civo-cluster-dns-record.png)

Specify the hostname in the Ingress template:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: fast-api
spec:
  rules:
  - host: k8s.crewsj.net
  http:
    paths:
      - path: /
        pathType: Exact
        backend:
          service:
            name: fast-api
            port:
              number: 80
```

After deploying the Deployment, Service and Ingress templates, I can get http://k8s.crewsj.net/

![FastAPI Result](https://s3.ap-southeast-2.amazonaws.com/blog.crewsj.net/shared_images/fast-api-result.png)