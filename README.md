# init-containers
Init containers in Kubernetes are specialized containers that run before the main containers in a Pod. They are primarily used for initialization or setup tasks, ensuring that certain conditions are met before the main application container starts.

# Multicontainer Pods
Before directly jumping into the init containers, let's discuss multicontainer Pods. A multicontainer Pod is a Pod that contains two or more containers that share resources, such as network, storage, or memory. These containers work together as a single unit to perform a specific task. Remember, the spec.containers field in a Pod specification is an array that allows you to define multiple containers:
'''
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
