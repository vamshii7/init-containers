# Contents
  1. Multicontainer Pods  
  2. Init containers  
  &nbsp;&nbsp;&nbsp;&nbsp;a. Benefits of init containers  
  &nbsp;&nbsp;&nbsp;&nbsp;b. Define an init container  
  3. Concluding  

# Multicontainer Pods
Before directly jumping into the init containers, let's discuss multicontainer Pods. A multicontainer Pod is a Pod that contains two or more containers that share resources, such as network, storage, or memory. These containers work together as a single unit to perform a specific task. Remember, the **spec.containers field** in a **Pod specification** is an array that allows you to define multiple containers:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: data
      mountPath: /var/log/nginx
  - name: logger
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/nginx/access.log']
    volumeMounts:
    - name: data
      mountPath: /var/log/nginx
  volumes:
  - name: data
    emptyDir: {}
```
Here, we defined two containers: app and logger. The app container runs the actual application, whereas the logger container uses the tail command to follow the nginx access log file. Both containers share a volume mount named data. Now, apply the YAML file.  

```Kubectl apply -f multi-container-pod.yaml```

![image](https://github.com/vamshii7/init-containers/assets/48650579/42b068ee-c1c7-4dad-8ac6-11f815ba895a)

# Init containers
As mentioned earlier, a Pod can comprise multiple containers. Similarly, you can define multiple init containers in a Pod under the spec.initContainers field, which is an array just like the containers field. If you define multiple init containers in a Pod, they run one after another in sequential order. Before starting the app container, all init containers must complete successfully. If any init container fails, the Pod will not launch.

# Benefits of init containers

**1. Small image size:** The init container uses independent images. This allows you to decouple dependencies and utilities from the actual app image, thereby reducing the size and complexity of the app image.  

**2. Dependency resolution:** Init containers run before the app containers start. This ensures that the app containers only start when all dependencies and preconditions are satisfied.  

**3. Reduced attack surface:** Keeping the external dependencies separate from the actual app image reduces the overall attack surface.  

**4. Security improvement:** Init containers can run with a different file system view than the app containers in the same Pod. They can now access Kubernetes Secrets that were previously inaccessible to app containers.  

# Define an init container

To understand an init container, see this Pod manifest file named webapp.yaml:

```apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  initContainers:
  - name: check-db
    image: busybox
    command: ['/bin/sh']
    args: ['-c', 'until nc -z db-service 3306; do echo waiting for db-service; sleep 5; done;']
  containers:
  - name: app
    image: harbor.testlab.local/library/webapp
    env:
    - name: DB_HOST
      value: db-service
    - name: DB_PORT
      value: "3306"
```

![image](https://github.com/vamshii7/init-containers/assets/48650579/497448a3-b186-4024-9732-64b0f8db3877)

In the screenshot, you can see a new spec.initContainers section, which is similar to the spec.containers section. We defined an init container named check-db, which uses a busybox image. Here, I used command and arguments to run the nc -z db-service 3306 command, which checks whether Port 3306 on the host "db-service" is open. The init container will not complete until it can reach db-service, and the app container waits for it to finish before starting.

Let's apply this YAML file and see what happens.

```
kubectl apply -f webapp.yaml
kubectl get pods webapp
```

![image](https://github.com/vamshii7/init-containers/assets/48650579/aa163451-0990-48a4-925b-4932c83b8fcb)

Notice how the Pod is stuck with a status of Init:0/1. This essentially means that 0 out of 1 init containers are complete so far. The kubectl describe pods command will tell you the state of the init and app containers, as shown in the screenshot below.

```kubectl describe pods webapp```

![image](https://github.com/vamshii7/init-containers/assets/48650579/f82a741f-8cdf-47da-bf7c-7dd033b46e08)

To see the container logs, you can run this command:

```kubectl logs webapp -c check-db```

![image](https://github.com/vamshii7/init-containers/assets/48650579/3746f0cf-dd6b-4a36-97d0-cec87e47840b)

You can see that the check-db init container is waiting for the db-service. Let's now create a YAML manifest to deploy the MySQL database in Kubernetes.

```
apiVersion: v1
data:
  password: UEBzc3cwcmQ=
kind: Secret
metadata:
  name: db-secret
---
apiVersion: v1
kind: Service
metadata:
  name: db-service
spec:
  ports:
  - port: 3306
  selector:
    app: db
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-deployment
spec:
  selector:
    matchLabels:
      app: db
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - image: mysql:5.7
        name: db-container
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
        ports:
        - containerPort: 3306
          name: db
        volumeMounts:
        - name: db-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: db-persistent-storage
        persistentVolumeClaim:
          claimName: db-pv-claim
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: db-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
This YAML file contains all the configurations required to create a Kubernetes Secret, Deployment, Service, persistent volume, and persistent volume claims. Let's apply this file and wait for the status of the webapp Pod to change.

```
kubectl apply -f database.yaml
kubectl get pods webapp --watch
```

![image](https://github.com/vamshii7/init-containers/assets/48650579/ae6e2cc8-a345-4fe6-874b-353ee5fe0a08)

You can see in the screenshot that when I created the db-service in the Kubernetes cluster, the webapp Pod started running because the check-db init container had completed.

![image](https://github.com/vamshii7/init-containers/assets/48650579/ebc621e3-4dbd-4183-b3d5-dd5fa6f5003c)

# Conclusion

You have learned about multicontainer Pods and completed the Kubernetes init containers demo. Note that this is just a simple use case of Kubernetes init containers. In a production environment, you will often encounter more complex start requirements for containers.
