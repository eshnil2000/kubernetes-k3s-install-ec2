# kubernetes-k3s-install-ec2

https://aws.amazon.com/premiumsupport/knowledge-center/eks-kubernetes-services-cluster/

worker- 18.191.20.41
master- 3.139.99.237
worker- 18.220.35.10

EXPOSING VIA NODEPORT ONLY EXPOSES SERVICE ON THE EXTERNAL IP ADDRESS OF THAT PARTICULAR NODE, WHERE THE POD IS DEPLOYED.  THIS IS NOT A LOADBALANCER.

Prepare 2 EC2 instance nodes using Pulumi
Open all inbound ports starting with port 80.
Open all outbound ports
Open port 22 to My IP Address only.

curl http://checkip.amazonaws.com
Install kubectl (Laptop) 
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

Install K3sup: https://github.com/eshnil2000/k3sup
K3SUP runs On Laptop: 
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/

//Make sure to set KUBECONFIG to point to where your kubeconfig file is saved. This points kubectl to the right cluster
export KUBECONFIG=`pwd`/kubeconfig

kubectl get node -o wide
kubectl describe node ip-172-31-32-231

export SERVER_IP=3.139.99.237
export USER=ubuntu
export AGENT_IP=18.223.15.59
export KUBE_EDITOR="/usr/bin/nano"

k3sup install --ip $SERVER_IP --user ubuntu  --ssh-key ~/keys/aws-poc.pem --k3s-extra-args '--disable traefik'

 k3sup join --ip $AGENT_IP --server-ip $SERVER_IP --user $USER --ssh-key ~/keys/aws-poc.pem  
 
k3sup join   --ip $NEXT_SERVER_IP   --user $USER   --server-user $USER   --server-ip $SERVER_IP --ssh-key ~/keys/aws-poc.pem

//ENABLE KUBERBETES DASHBOARD
GITHUB_URL=https://github.com/kubernetes/dashboard/releases
VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
//CREATE SERVICE ACCOUNT AND ROLE
admine-service-user.yaml
//https://blog.tekspace.io/deploying-kubernetes-dashboard-in-k3s-cluster/
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
//create profile admn-user-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
  
 
///
kubectl create -f admin-user-role.yaml -f admin-service-user.yaml

kubectl expose deployment kubernetes-dashboard  --type=NodePort  --name=kubernetes-dashboard-nodeport -n kubernetes-dashboard

    kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard
In edit mode change type: ClusterIP to type: NodePort. And Save it.

kubectl -n kubernetes-dashboard describe secret admin-user-token

browse to node IP address, but use https://18.191.20.41:31127/#/role?namespace=_all

cat <<EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF

kubectl apply -f nginx-deployment.yaml
kubectl get pods -l 'app=nginx' -o wide | awk {'print $1" " $3 " " $6'} | column -t


kubectl expose deployment nginx-deployment  --type=NodePort  --name=nginx-service-nodeport
//At this point, the service is available externally : http://3.139.99.237:31691/

kubectl get pods -l 'app=nginx' -o wide | awk {'print $1" " $3 " " $6'} | column -t
kubectl get nodes -o wide |  awk {'print $1" " $2 " " $7'} | column -t

kubectl get pods nginx-deployment-66b6c48dd5-dn75x -o yaml

//get all port forwards
kubectl get svc --all-namespaces -o go-template='{{range .items}}{{range.spec.ports}}{{if .nodePort}}{{.nodePort}}{{"\n"}}{{end}}{{end}}{{end}}'

kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName --all-namespaces

kubectl get pods --all-namespaces -o jsonpath="{..image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c

TO CREATE AND RUN A CONTAINER IMAGE:
build an image with app running on port 8080

//
file: app.js
const http = require('http');
const os = require('os');
console.log("Kubia server starting...");
var handler = function(request, response) {
console.log("Received request from " + request.connection.remoteAddress);
response.writeHead(200);
response.end("You've hit " + os.hostname() + "\n");
};
var www = http.createServer(handler);
www.listen(8080);
//

//file Dockerfile
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
//
sudo docker build -t kubia .
docker tag kubia eshnil2000/kubia
//test docker run -p 8080:8080 -d eshnil2000/kubia

kubectl run kubia --image=eshnil2000/kubia --port=8080

//Launching app from kubernetes
//contents of kubia.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
  labels:
    app: kubia
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: jwilder/whoami
        ports:
        - containerPort: 8000
          protocol: TCP
//end kubia.yaml


kubectl apply -f kubia.yaml
kubectl expose deployment kubia  --type=NodePort  --name=kubia-http

//THIS SERVIE IS AVAILABLE ONLY ON THE NODE EXTERNAL IP ADDRESS WHERE THIS APP IS DEPLOYED, if container happens to launch on another node, you cant access it. TO FORCE IT TO DEPLOY TO MASTER, SCALE THE SERVICE

kubectl scale --replicas=10 deployment/kubia

//Replication Controller
kubectl create -f kubia-rc.yaml
kubectl label pod kubia-rzppr app=foo --overwrite
kubectl expose rc kubia --type=NodePort  --name=kubia-rc
kubectl get pods -L app
kubectl get pods --show-labels
kubectl scale rc kubia --replicas=10
//delete the rc but keep the existing pods running
kubectl delete rc kubia --cascade=false
//kubia-rc.yaml

replicationcontroller "kubia" created
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 5
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        ports:
        - containerPort: 8080

//CURL ONE CONTAINER POD FROM ANOTHER
kubectl exec kubia-gv9gj -- curl -s http://10.43.43.137


//to enable ingress:
//1. launch a replication controller
kubectl apply -f nginx-rc.yaml
//2. launch a service
kubectl apply -f nginx-svc-nodeport.yaml
//3. launch an ingress resource (since ingress controller traefik is already running)
kubectl apply -f nginx-ingress.yaml

//Since nginx runs on root path only (/), need to strip prefix in traefik rules:
////nginx-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
//end example pathprefixstrip
//nginx-rc.yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

//nginx-svc-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30125
  selector:
    app: nginx

//nginx-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubia
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.frontend.rule.type: PathPrefixStrip
spec:
  rules:
  - host: www.codenovator.net
    http:
      paths:
      - path: /
        backend:
          serviceName: whoami-nodeport
          servicePort: 8000
  - host: nginx.codenovator.net
    http:
      paths:
      - path: /nginx
        backend:
          serviceName: nginx-nodeport
          servicePort: 80
//READINESS PROBE - doesn't route traffic to the container till it's ready
apiVersion: v1
kind: ReplicationController
metadata:
  name: whoami
spec:
  replicas: 2
  selector:
    app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
      - name: whoami
        image: jwilder/whoami
        ports:
        - containerPort: 8000
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
//
//ENABLE TRAEFIK v2.4
//on master node, uninstall k3s
/usr/local/bin/k3s-uninstall.sh
//disable default traefik
k3sup install --ip $SERVER_IP --user ubuntu  --ssh-key ~/keys/aws-poc.pem --k3s-extra-args '--disable traefik'

//follow
https://doc.traefik.io/traefik/user-guides/crd-acme/#:~:text=Traefik%20with%20an%20IngressRoute%20Custom,TLS%20setup%20with%20Let's%20Encrypt.

kubectl 
apply -f TRAEFIK-MANUAL-INSTALL/crd-traefik-ingress.yaml
kubectl apply -f TRAEFIK-MANUAL-INSTALL/svc-whoami.yaml
kubectl apply -f TRAEFIK-MANUAL-INSTALL/deploy-traefik-whoami.yaml

//on leader node 
sudo kubectl port-forward --address 0.0.0.0 service/traefik 8000:8000 8080:8080 443:4443 -n default

kubectl apply -f ingress-routes.yaml

//ingress-routes.yaml
```
 apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: simpleingressroute
  namespace: default
spec:
  entryPoints:
    - web
  routes:
  - match: Host(`www.codenovator.net`) && PathPrefix(`/notls`)
    kind: Rule
    services:
    - name: whoami
      port: 80

---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: ingressroutetls
  namespace: default
spec:
  entryPoints:
    - websecure
  routes:
  - match: Host(`www.codenovator.net`) && PathPrefix(`/tls`)
    kind: Rule                                                             services:
    - name: whoami
      port: 80
  tls:
    certResolver: myresolver  
```
=====

curl [-k] https://your.example.com/tls

curl http://your.example.com:8000/notls

//dashboard
http://www.codenovator.net:8080/dashboard/#/http/routers
 
// to set up LetsEncrypt :
 https://bitbucket.org/eshnil2000/server-setup/src/master/docker-compose.traefik-labs.yml
 ```
 #docker network create -d overlay --attachable traefik_default
# save sensitive variables in .env file
#validate config using : docker-compose -f docker-compose.traefik-labs.yml config
# env $(cat .env | grep ^[A-Z] | xargs) docker stack deploy -c docker-compose.traefik-labs.yml traefik
version: '3'
networks:
  traefik_default:
    driver: overlay
    external:
      name:  traefik_default
services:
  traefik:
    # The latest official supported Traefik docker image
    image: traefik:v2.3
    # Enables the Traefik Dashboard and tells Traefik to listen to docker
    # enable --log.level=INFO so we can see what Traefik is doing in the log files
    ports:
      # Exposes port 80 for incomming web requests
      - "80:80"
      - "443:443"
      # The Web UI port http://0.0.0.0:8080 (enabled by --api.insecure=true)
      #- "8080:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock
      # Copies the Let's Encrypt certificate locally for ease of backing up
      - /letsencrypt:/letsencrypt
       # Mounts the Traefik static configuration inside the Traefik container
      - ./traefik.dns.yml:/etc/traefik/traefik.yml
    
    networks:
      traefik_default:

    environment:
      - "NAMECOM_API_TOKEN=${NAMECOM_API_TOKEN}"
      - "NAMECOM_USERNAME=${NAMECOM_USERNAME}"
    deploy:
      labels:
        - "traefik.http.routers.traefik.rule=Host(`traefik.${DOMAIN_NAME}`)"
        - "traefik.http.routers.traefik-secure.rule=Host(`traefik.${DOMAIN_NAME}`)"
        - "traefik.enable=true"
        - "traefik.http.routers.traefik.service=api@internal"
        - "traefik.http.routers.traefik.entrypoints=web"
        - "traefik.http.routers.traefik-secure.entrypoints=websecure"
        - "traefik.http.routers.traefik-secure.tls.certresolver=myresolver" 
        - "traefik.http.routers.traefik-secure.tls.domains[0].main=${DOMAIN_NAME}"
        - "traefik.http.routers.traefik-secure.tls.domains[0].sans=*.${DOMAIN_NAME}"
        - "traefik.http.routers.traefik-secure.tls=true"
        - "traefik.http.routers.traefik-secure.service=api@internal"
        - "traefik.http.routers.traefik.middlewares=test-auth"
        - "traefik.http.routers.traefik-secure.middlewares=test-auth"
        - "traefik.http.middlewares.test-auth.basicauth.users=${ADMIN_HASH}"
        - "traefik.http.services.noop.loadbalancer.server.port=8080"
        #- "traefik.docker.lbswarm=true"
        # Create hash password -> echo $(htpasswd -nb user2 test123) | sed -e s/\\$/\\$\\$/g
        #EXAMPLE: echo $(htpasswd -nb test test) | sed -e s/\\$/\\$\\$/g
        #RESULT: test:$$apr1$$yNcnWSIn$$nDdzj/Uwufk9VdzBlRTFh/
        #EXAMPLE IF LOADING FROM .env file: echo $(htpasswd -nb test test)
      
```
 * contents of .env
 ```
NAMECOM_API_TOKEN=xxxx
NAMECOM_USERNAME=xxx
DOMAIN_NAME=dappsuni.com
 ```
