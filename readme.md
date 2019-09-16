# readme.md

## Notes while working through <https://aksworkshop.io>

See also the kubernetes walkthrough <https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough>

## Deploy k8s with aks

List latest k8s versions available in AKS

        az aks get-versions -l eastus --output table

Get the latest version of k8s in eastus region

        version=$(az aks get-versions -l eastus --query 'orchestrators[-1].orchestratorVersion' -o tsv)

Create a resource group (akswsrg)

        az group create -n akswsrg -l eastus

Create a service principal for resource interactions. Make a note of the appID and password returned in the json

    az ad sp create-for-rbac --skip-assignment

Create an aks cluster with monitoring enabled (takes a couple of minutes). The kubernetes-version is the newest non-preview version. Found it using *az aks get-versions -l eastus --output table*.
*Note: The service-principle and --client-secret params must be used. If you don't run the above command you'll get the following error when you attenpt to create the aks cluster.*

Operation failed with status: 'Bad Request'. Details: Service principal clientID: <guid> not found in Active Directory tenant <guid>, Please see https://aka.ms/aks-sp-help for more details.

Wait a couple of minutes before running the command below. Give the service principal a chance to propogate. The command will take several minutes to complete.

        az aks create --resource-group akswsrg --name blizzakscluster02 --node-count 2 --enable-addons monitoring --generate-ssh-keys --kubernetes-version 1.14.6 --service-principal <appID> --client-secret <password>

Check the status of the cluster, to see if it has been created. (You can free up your terminal from the above command while it still runs in the background via CTRL-C.)

        az aks list --output table

Configure kubectl to connect to the cluster, then verify the connection by listing the nodes and pods. (There shouldn't be any pods yet, but it's fun to check anyway.)

        az aks get-credentials --resource-group akswsrg --name blizzakscluster02

        kubectl get nodes
        kubectl get pods

## deploy MongoDb

Be careful with the authentication settings when creating MongoDB. It is recommended that you create a standalone username/password and database.

Important: If you install using Helm and then delete the release, the MongoDB data and configuration persists in a Persistent Volume Claim. You may face issues if you redeploy again using the same release name because the authentication configuration will not match. If you need to delete the Helm deployment and start over, make sure you delete the Persistent Volume Claims created otherwise you’ll run into issues with authentication due to stale configuration. Find those claims using *kubectl get pvc*

Deploy an instance of mongoDB to your cluster. The database name should be *akschallenge*. Do not modify it.

Install helm using chocolatey, since it's not already on my laptop. Need to be in an elevated command shell.

Note: no need to run this command if helm is already installed. (duh)

        choco install kubernetes-helm

Create file *helm-rbac.yaml* with the following contents

        apiVersion: v1
        kind: ServiceAccount
        metadata:
        name: tiller
        namespace: kube-system
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
        name: tiller
        roleRef:
        apiGroup: rbac.authorization.k8s.io
        kind: ClusterRole
        name: cluster-admin
        subjects:
        - kind: ServiceAccount
            name: tiller
            namespace: kube-system

Deploy it using

        kubectl apply -f helm-rbac.yaml

Initialize tiller

        helm init --service-account tiller

Output

        $HELM_HOME has been configured at C:\Users\jimbl\.helm.

        Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

        Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
        To prevent this, run `helm init` with the --tiller-tls-verify flag.
        For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation

Install the MongoDB helm chart

        helm install stable/mongodb --name orders-mongo --set mongodbUsername=<orders-user>,mongodbPassword=<orders-password>,mongodbDatabase=akschallenge

Output

        NAME:   orders-mongo
        LAST DEPLOYED: Sun Sep 15 14:42:06 2019
        NAMESPACE: default
        STATUS: DEPLOYED

        RESOURCES:
        ==> v1/PersistentVolumeClaim
        NAME                  STATUS   VOLUME   CAPACITY  ACCESS MODES  STORAGECLASS  AGE
        orders-mongo-mongodb  Pending  default  1s

        ==> v1/Pod(related)
        NAME                                   READY  STATUS   RESTARTS  AGE
        orders-mongo-mongodb-797b6b4f7d-mmnqm  0/1    Pending  0         1s

        ==> v1/Secret
        NAME                  TYPE    DATA  AGE
        orders-mongo-mongodb  Opaque  2     1s

        ==> v1/Service
        NAME                  TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)    AGE
        orders-mongo-mongodb  ClusterIP  10.0.90.27  <none>       27017/TCP  1s

        ==> v1beta1/Deployment
        NAME                  READY  UP-TO-DATE  AVAILABLE  AGE
        orders-mongo-mongodb  0/1    1           0          1s


        NOTES:


        ** Please be patient while the chart is being deployed **

        MongoDB can be accessed via port 27017 on the following DNS name from within your cluster:

            orders-mongo-mongodb.default.svc.cluster.local

        To get the root password run:

            export MONGODB_ROOT_PASSWORD=$(kubectl get secret --namespace default orders-mongo-mongodb -o jsonpath="{.data.mongodb-root-password}" | base64 --decode)

        To get the password for "blizz" run:

            export MONGODB_PASSWORD=$(kubectl get secret --namespace default orders-mongo-mongodb -o jsonpath="{.data.mongodb-password}" | base64 --decode)

        To connect to your database run the following command:

            kubectl run --namespace default orders-mongo-mongodb-client --rm --tty -i --restart='Never' --image bitnami/mongodb --command -- mongo admin --host orders-mongo-mongodb --authenticationDatabase admin -u root -p $MONGODB_ROOT_PASSWORD

        To connect to your database from outside the cluster execute the following commands:

            kubectl port-forward --namespace default svc/orders-mongo-mongodb 27017:27017 &
            mongo --host 127.0.0.1 --authenticationDatabase admin -p $MONGODB_ROOT_PASSWORD

Create a k8s secret to hold the mongoDB details. This includes the mongoHost, mongoUser, and mongoPassword.

        kubectl create secret generic mongodb --from-literal=mongoHost="orders-mongo-mongodb.default.svc.cluster.local" --from-literal=mongoUser="<orders-user>" --from-literal=mongoPassword="<orders-password>"

Hint By default, the service load balancing the MongoDB cluster would be accessible at orders-mongo-mongodb.default.svc.cluster.local

You’ll need to use the user created in the command above when configuring the deployment environment variables.

## deploy the order capture api

Set environment variables

        MONGOHOST="<hostname of mongodb>"
        MongoDB hostname. Read from a Kubernetes secret called mongodb.
        MONGOUSER="<mongodb username>"
        MongoDB username. Read from a Kubernetes secret called mongodb.
        MONGOPASSWORD="<mongodb password>"
        MongoDB password. Read from a Kubernetes secret called mongodb.

Hint: The Order Capture API exposes the following endpoint for health-checks once you have completed the tasks below: http://[PublicEndpoint]:[port]/healthz

Provision the *captureorder* deployment and expose a public endpoint. Save the following as *captureorder-deployment.yaml*

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: captureorder
        spec:
        selector:
            matchLabels:
                app: captureorder
        replicas: 2
        template:
            metadata:
                labels:
                    app: captureorder
            spec:
                containers:
                - name: captureorder
                image: azch/captureorder
                imagePullPolicy: Always
                readinessProbe:
                    httpGet:
                    port: 8080
                    path: /healthz
                livenessProbe:
                    httpGet:
                    port: 8080
                    path: /healthz
                resources:
                    requests:
                    memory: "128Mi"
                    cpu: "100m"
                    limits:
                    memory: "256Mi"
                    cpu: "500m"
                env:
                - name: MONGOHOST
                    valueFrom:
                    secretKeyRef:
                        name: mongodb
                        key: mongoHost
                - name: MONGOUSER
                    valueFrom:
                    secretKeyRef:
                        name: mongodb
                        key: mongoUser
                - name: MONGOPASSWORD
                    valueFrom:
                    secretKeyRef:
                        name: mongodb
                        key: mongoPassword
                ports:
                - containerPort: 8080

Deploy it using

        kubectl apply -f captureorder-deployment.yaml

Verify the pods are up and running

        kubectl get pods -l app=captureorder -w

        NAME                           READY   STATUS             RESTARTS   AGE
        captureorder-894bbf6d7-7qmv4   0/1     CrashLoopBackOff   3          103s
        captureorder-894bbf6d7-k4zq6   0/1     CrashLoopBackOff   3          103s

Nope. Crashing with CrashLoopBackoff. Troubleshoot here <https://managedkube.com/kubernetes/pod/failure/crashloopbackoff/k8sbot/troubleshooting/2019/02/12/pod-failure-crashloopbackoff.html>

I should not have used a $ in the password. It seems to have grabbed some random value and plugged it in.

Troubleshooting Pending Pods <https://managedkube.com/kubernetes/k8sbot/troubleshooting/pending/pod/2019/02/22/pending-pod.html>

If you want to wipe out the pods and not have them return, you need to *delete the deployment*

        kubectl get all
        NAME                               READY   STATUS    RESTARTS   AGE
        pod/captureorder-894bbf6d7-vg8st   0/1     Pending   0          5m25s
        pod/captureorder-894bbf6d7-xps69   0/1     Pending   0          10s

        NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
        service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   59m

        NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
        deployment.apps/captureorder   0/2     2            0           18m

        NAME                                     DESIRED   CURRENT   READY   AGE
        replicaset.apps/captureorder-894bbf6d7   2         2         0       18m
        
        
        kubectl delete deployment.apps/captureorder
        deployment.apps "captureorder" deleted
        

        kubectl get all
        NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
        service/kubernetes   ClusterIP   10.0.0.1     <none>        443/TCP   60m

I ended up having to delete everything: the deployment the service principal, and the resource group. Then I started from the top of the instructions. Second time through worked fine. I updated the notes a bit as I went through it the second time.

And everything seems to be running now.

        kubectl get pods -l app=captureorder -w
        NAME                           READY   STATUS    RESTARTS   AGE
        captureorder-894bbf6d7-tqlwj   1/1     Running   0          17s
        captureorder-894bbf6d7-x29vn   1/1     Running   0          17s

Create the Service definition *captureorder-service.yaml*

    apiVersion: v1
    kind: Service
    metadata:
    name: captureorder
    spec:
    selector:
        app: captureorder
    ports:
    - protocol: TCP
        port: 80
        targetPort: 8080
    type: LoadBalancer

And deploy it using

    kubectl apply -f captureorder-service.yaml

Give it a couple of minutes, then retrieve the external-ip of the service

    kubectl get service captureorder -o jsonpath="{.status.loadBalancer.ingress[*].ip}" -w

Ensure orders are successfully written to MongoDB

Hint: You can test your deployed API either by using Postman or Swagger with the following endpoint : http://[Your Service Public LoadBalancer IP]/swagger/

Send a POST request using CURL

        curl -d '{"EmailAddress": "billy@bob.com", "Product": "prod-1", "Total": 100}' -H "Content-Type: application/json" -X POST http://13.92.154.111/v1/order

It works

         curl -d '{"EmailAddress": "billy@bob.com", "Product": "prod-1", "Total": 100}' -H "Content-Type: application/json" -X POST http://13.92.154.111/v1/order

        {
        "orderId": "5d7e8e0b0dd320000187ff5e"
        }

## deploy the frontend using Ingress

Provision the frontend deployment

Save the *frontend* deployment definition as *frontend-deployment.yaml*

        apiVersion: apps/v1
        kind: Deployment
        metadata:
        name: frontend
        spec:
        selector:
            matchLabels:
                app: frontend
        replicas: 1
        template:
            metadata:
                labels:
                    app: frontend
            spec:
                containers:
                - name: frontend
                image: azch/frontend
                imagePullPolicy: Always
                readinessProbe:
                    httpGet:
                    port: 8080
                    path: /
                livenessProbe:
                    httpGet:
                    port: 8080
                    path: /
                resources:
                    requests:
                    memory: "128Mi"
                    cpu: "100m"
                    limits:
                    memory: "256Mi"
                    cpu: "500m"
                env:
                - name: CAPTUREORDERSERVICEIP
                    value: "13.92.154.111"  # Replace with your captureorder service IP
                ports:
                - containerPort: 8080

Deploy it using this

        kubectl apply -f frontend-deployment.yaml

Verify the pods are up and running

        kubectl get pods -l app=frontend -w

        NAME                        READY   STATUS    RESTARTS   AGE
        frontend-8576546794-ks87s   1/1     Running   0          115s

Exponse the frontend on a hostname

We're going to use the nginx-ingress controller.

Here's the frontend-service.yaml file

        apiVersion: v1
        kind: Service
        metadata:
        name: frontend
        spec:
        selector:
            app: frontend
        ports:
        - protocol: TCP
            port: 80
            targetPort: 8080    
        type: ClusterIP

Deploy it using

        kubectl apply -f frontend-service.yaml

Deploy the ingress controller using helm

        helm repo update

        helm upgrade --install ingress stable/nginx-ingress --namespace ingress

It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running

        kubectl --namespace ingress get services -o wide -w ingress-nginx-ingress-controller

In a couple of minutes a public IP address will be allocated to the ingress controller. Retrieve with

        kubectl get svc -n ingress ingress-nginx-ingress-controller -o jsonpath="{.status.loadBalancer.ingress[*].ip}"

Create an ingress resource. Use the public IP address for the controller, and the serviceName and servicePort point to the service you deployed previously. Save the file as *frontend-ingress.yaml*

        apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
        name: frontend
        spec:
        rules:
        - host: frontend.40.85.180.205.nip.io
            http:
            paths:
            - backend:
                serviceName: frontend
                servicePort: 80
                path: /

Create it using

        kubectl apply -f frontend-ingress.yaml

Browse to the publich hostname of the frontend and watch as the number of orders change. Give it a couple of minutes if it doesn't work from the first trial. You might also need to enable cross-scripting in your browser (in chrome, allow unsafe script to be executed).

        http://frontend.40.85.180.205.nip.io

It works.

## Enable SSL/TLS on ingress

We'll use the *Let's Encrypt* free service <https://letsencrypt.org/>.

Important After you finish this task for the frontend, you may either receive some browser warnings about “mixed content” or the orders might not load at all because the calls happen via JavaScript. Use the same concepts to create an ingress for captureorder service and use SSL/TLS to secure it in order to fix this.

Install *cert-manager* (a k8s add-on) using helm and configure it to use *letsencrypt* as the cert issuer.

        helm install stable/cert-manager --name cert-manager --set ingressShim.defaultIssuerName=letsencrypt --set ingressShim.defaultIssuerKind=ClusterIssuer --version v0.5.2

Create a let's encrypt clusterIssuer. Save the following as *letsencrypt-clusterissuer.yaml*

        apiVersion: certmanager.k8s.io/v1alpha1
        kind: ClusterIssuer
        metadata:
        name: letsencrypt
        spec:
        acme:
            server: https://acme-v02.api.letsencrypt.org/directory # production
            #server: https://acme-staging-v02.api.letsencrypt.org/directory # staging
            email: _YOUR_EMAIL_ # replace this with your email
            privateKeySecretRef:
            name: letsencrypt
            http01: {}

And apply it using

        kubectl apply -f letsencrypt-clusterissuer.yaml

Update the ingress resource to auto request a cert. Save the following as *frontend-ingress-tls.yaml*

Note: Make sure to replace _INGRESS_CONTROLLER_EXTERNAL_IP_ with your cluster ingress controller external IP. Also make note of the secretName: frontend-tls-secret as this is where the issued certificate will be stored as a Kubernetes secret.

        apiVersion: extensions/v1beta1
        kind: Ingress
        metadata:
        name: frontend
        annotations:
            certmanager.k8s.io/cluster-issuer: letsencrypt
        spec:
        tls:
        - hosts:
            - frontend._INGRESS_CONTROLLER_EXTERNAL_IP_.nip.io
            secretName: frontend-tls-secret
        rules:
        - host: frontend._INGRESS_CONTROLLER_EXTERNAL_IP_.nip.io
            http:
            paths:
            - backend:
                serviceName: frontend
                servicePort: 80
                path: /

Apply it using

        kubectl apply -f frontend-ingress-tls.yaml

Verify the cert is issued and test the website over SSL. Make sure the cert has been issued by running

        kubectl describe certificate frontend

Output

    Failed to create new order: acme: urn:ietf:params:acme:error:rateLimited: Error creating new order :: too many certificates already issued for: nip.io: see https://letsencrypt.org/docs/rate-limits/

Ah well, we'll just move on.

## Monitoring

Create a Log Analytics workspace via the CLI by following these instructions <https://docs.microsoft.com/en-us/azure/azure-monitor/learn/quick-create-workspace-cli>

Save the template as *deploylaworkspacetemplate.json*

Deploy using

        az group deployment create --resource-group akswsrg --name blizzLAWorkspace  --template-file deploylaworkspacetemplate.json

It can take a few minutes to complete. 

It looks as though the monitoring addon is automatically enabled with this command, so you probably don't have to run the following command. Otherwise, enable the monitoring add-on (see the <http://aksworkshop.io> instructions) by running

        az aks enable-addons --resource-group akswsrg --name blizzakscluster02 --addons monitoring --workspace-resource-id /subscriptions/<subscription-id>/resourcegroups/<resource-group>/providers/microsoft.operationalinsights/workspaces/<workspace-name>

But it looks as though it's not necessary to run the above command. I had to disable the addon before I could run it. The disable command I used was

        az aks disable-addons -a monitoring -g akswsrg -n blizzakscluster02

Then I re-ran

    az aks enable-addons --resource-group akswsrg --name blizzakscluster02 --addons monitoring --workspace-resource-id /subscriptions/<subscription-id>/resourcegroups/<resource-group>/providers/microsoft.operationalinsights/workspaces/<workspace-name>

Leverage integrated AKS monitoring to figure out if requests are failing, inspect k8s event or logs, and monitor you cluster health. Do this from the Azure portal. Go to the resource group > k8s cluster > Monitor containers

View the live container logs and k8s events

Create *logreader-rbac.yaml*

Deploy it using

        kubectl apply -f logreader-rbac.yaml

If you have a Kubernetes cluster that is not configured with Kubernetes RBAC authorization or integrated with Azure AD single-sign on, you do not need to follow the steps above. Because Kubernetes authorization uses the kube-api, contributor access is required.

Head over to the AKS cluster on the Azure portal, click on Insights under Monitoring, click on the Containers tab and pick a container to view its live logs or event logs and debug what is going on.

I skipped the Prometheus metrics task.

## Scaling

Run a baseline test

There is a a container image on Docker Hub (azch/loadtest) that is preconfigured to run the load test. You may run it in Azure Container Instances running the command below

        az container create -g akswsrg -n loadtest --image azch/loadtest --restart-policy Never -e SERVICE_IP=frontend._INGRESS_CONTROLLER_EXTERNAL_IP_.nip.io

You may view the logs of the Azure Container Instance streaming logs by running the command below. You may need to wait for a few minutes to get the full logs, or run this command multiple times.

        az container logs -g akswsrg -n loadtest

When you’re done, you may delete it by running

        az container delete -g akswsrg -n loadtest

You may use the Azure Monitor (previous task) to view the logs and figure out where you need to optimize to increase the throughtput (requests/sec), reduce the average latency and error count.

I skipped the rest of the performance section.

### Create an ACR azure container registry

Run these commands to create the RG and ACR

        az acr login

        az group create -n acrrg -l eastus

        az acr create --resource-group acrrg --name acrForBlizz --sku Standard --location eastus

Run this to build and push the container images

        az acr build -t "captureorder:{{.Run.ID}}" -r acrForBlizz .

Note You’ll get a build ID in a message similar to Run ID: ca3 was successful after 3m14s. Use ca3 in this example as the image tag in the next step.

        ca1: digest: sha256:133469ab3a14ad73eab6acf431ded6531a0b3fbb7508ea04370a869347326fe4 size: 949
        2019/09/15 21:31:18 Successfully pushed image: acrforblizz.azurecr.io/captureorder:ca1
        2019/09/15 21:31:18 Step ID: build marked as successful (elapsed time in seconds: 111.007041)
        2019/09/15 21:31:18 Populating digests for step ID: build...
        2019/09/15 21:31:21 Successfully populated digests for step ID: build
        2019/09/15 21:31:21 Step ID: push marked as successful (elapsed time in seconds: 3.629561)
        2019/09/15 21:31:21 The following dependencies were found:
        2019/09/15 21:31:21
        - image:
            registry: acrforblizz.azurecr.io
            repository: captureorder
            tag: ca1
            digest: sha256:133469ab3a14ad73eab6acf431ded6531a0b3fbb7508ea04370a869347326fe4
        runtime-dependency:
            registry: registry.hub.docker.com
            repository: library/alpine
            tag: latest
            digest: sha256:72c42ed48c3a2db31b7dafe17d275b634664a708d901ec9fd57b1529280f01fb
        buildtime-dependency:
        - registry: registry.hub.docker.com
            repository: library/golang
            tag: 1.9.4
            digest: sha256:332a39d7995bc6770e70cc9c48bae1e5724d91f88c62856ffe3b32f64419622d
        git: {}

        Run ID: ca1 was successful after 2m3s

Configure your application to pull from your private registry

Grant AKS generated service principal to ACR. Since I'm using pwsh, I needed to put a $ in front of the vars and quotes around the strings

        $AKS_RESOURCE_GROUP="akswsrg"
        $AKS_CLUSTER_NAME="blizzakscluster02"
        $ACR_RESOURCE_GROUP="acrrg"
        $ACR_NAME="acrForBlizz"

        # Get the id of the service principal configured for AKS
        $CLIENT_ID=$(az aks show --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" --output tsv)

        # Get the ACR registry resource id
        $ACR_ID=$(az acr show --name $ACR_NAME --resource-group $ACR_RESOURCE_GROUP --query "id" --output tsv)

        # Create role assignment
        az role assignment create --assignee $CLIENT_ID --role acrpull --scope $ACR_ID

After you grant your Kubernetes cluster access to your private registry, you can update your deployment with the image you built in the previous step.

Kubernetes is declarative and keeps a manifest of all object resources. Edit your deployment object with the updated image.

Update your deployment object with the updated image

        kubectl edit deployment captureorder

Replace the image tag with the location of the new image on Azure Container Registry. Replace <build id> with the ID you got from the message similar to Run ID: ca3 was successful after 3m14s after the build was completed.

Hint kubectl edit launches vim. To search in vim you can type /image: azch/captureorder. Go into insert mode by typing i, and replace the image with <unique-acr-name>.azurecr.io/captureorder:<build id>

Quit the editor by hitting Esc then typing :wq and run kubectl get pods -l app=captureorder -w.

If you successfully granted Kubernetes authorization to your private registry you will see one pod terminating and a new one creating. If the access to your private registry was properly granted to your cluster, your new pod should be up and running within 10 seconds.

# DevOps tasks

## CI and CD

Hint The source code repositories on GitHub contain an azure-pipelines.yml definition that you can use with Azure Pipelines to build and deploy the containers. See <https://github.com/Azure/azch-captureorder/blob/master/azure-pipelines.yml>
