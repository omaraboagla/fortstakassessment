**Install and Start K3s Cluster:

Tried installing and starting K3s Cluster, was facing errors and startup breakdowns, after hours of going through forums and using AI Models to fix it, I came into a conclusion that K3s need a full operating Linux environemtn to run, and that since I'm working with WSL2, there are some files that it needs to communicate with that aren't present. Alternatively, I went with K3d which is a K3s-based Kubernetes cluster, that runs K3s inside Docker.

**Install and Start K3d Cluster:

k3d cluster create todo-cluster --agents 1

**Verified it:

kubectl get nodes


**Since K3d runs in its own isolated Docker environment, I had to import my locally built image into it, so I built and imported the Todo App Docker Image:

docker build -t todo-app:latest .
k3d image import todo-app:latest -c todo-cluster

**I created 2 essential K8s YAMLs, deployment & service:

nano todo-deployment.yaml
(Defines the app deployment and ensures MongoDB uses the correct local port 27018.
env:
  - name: MONGO_URL
    value: mongodb://host.k3d.internal:27018)
ports:
  - containerPort: 4001

nano todo-service.yaml
(Created a service exposing the app on port 4000.
ports:
  - protocol: TCP
    port: 4001
    targetPort: 4001)

**Created a pod and the associated service to deploy app to K3s:

kubectl apply -f todo-deployment.yaml
kubectl apply -f todo-service.yaml

**Initially, pods were stuck in (STATUS: ErrImagePull / ImagePullBackOff
This was because Kubernetes couldn't find the image in a public registry) To resolve this, I:

Imported the image manually using k3d image import.
Deleted the failed pod using:
kubectl delete pod -l app=todo

**Once restarted, the new pod successfully picked up the local image.

kubectl get pods
kubectl get svc

**I exposed the service to my local machine by forwarding it to port 4001, since port 4000 was already in use by the same app image running directly on the VM from Part 3.:

kubectl port-forward service/todo-service 4001:4001

**Tested http://localhost:4001:

Web app works successfully!


**Deploying and GitOps the app with k3d + ArgoCD:

**Installed ArgoCD in my K3s cluster:

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

**Exposed ArgoCD UI locally. Since k3d doesn’t expose LoadBalancer by default, I'll port-forward the ArgoCD API server:

kubectl port-forward svc/argocd-server -n argocd 8080:443

**Checked ArgoCD:

http:localhost:8080

**Directory already looks like this on WSL2:
~/todo-deployment/
├── todo-deployment.yaml
└── todo-service.yaml

**Created kustomization.yaml file (for ArgoCD):

Contains:

cd ~/todo-deployment

cat <<EOF > kustomization.yaml
resources:
  - todo-deployment.yaml
  - todo-service.yaml
EOF

**Created argo-app.yaml in the same ~/todo-deployment folder:

Content:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: todo-app
  namespace: argocd
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  project: default
  source:
    repoURL: file:///home/devops/todo-deployment
    targetRevision: HEAD
    path: .
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

**Applied ArgoCD Application:

kubectl apply -f argo-app.yaml

returned:
application.argoproj.io/todo-app created

checked & watched status:
kubectl get applications -n argocd
kubectl describe application todo-app -n argocd

**Refreshed ArgoCD UI on https://localhost:8080/:

**Got default admin password by:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

returned:
 jsonpath="{.data.password}" | base64 -d && echo
gsCjdDT7Kb3hkElv

**logged in by:
Username: admin
Password: gsCjdDT7Kb3hkElv

**In the ArgoCD UI, checked for:

1. The todo-app application listed
2.A green “Synced” and “Healthy” state.
3.Clicked the app to see the todo-deployment and todo-service.