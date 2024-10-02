Step 1: Log in to the OpenShift Cluster: Log in to your OpenShift cluster using the oc CLI.

```oc login --token=<your-token> --server=<your-server>```

```oc new-project splunk```

Step 2: Create deploymnet without PV and PVC 

```oc create secret generic splunk-secret --from-literal=SPLUNK_PASSWORD='your_password'```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: splunk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: splunk
  template:
    metadata:
      labels:
        app: splunk
    spec:
      containers:
      - name: splunk
        image: splunk/splunk:latest     # Use the latest or your specific version of Splunk
        ports:
        - containerPort: 8000           # Splunk Web Interface
        - containerPort: 8089           # Splunk Management Port
        - containerPort: 9997           # Splunk Forwarding Port
        env:
        - name: SPLUNK_START_ARGS
          value: "--accept-license --answer-yes"
        - name: SPLUNK_PASSWORD         # Retrieve the password from the secret
          valueFrom:
            secretKeyRef:
              name: splunk-secret
              key: SPLUNK_PASSWORD
      securityContext:
        runAsUser: 41812                # The Splunk user inside the container
        fsGroup: 41812                  # Group that has write access
```

Step 2: Create Persistent Volumes (PV) and Persistent Volume Claims (PVC)
Define a Persistent Volume (PV) for Splunk storage: Splunk requires persistent storage to store indexed data. Create a persistent volume YAML file (adjust for your storage type, e.g., NFS, OCS, etc.).

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: splunk-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /path/to/nfs/share
    server: nfs-server-ip
```

```oc apply -f splunk-pv.yaml```

Create a Persistent Volume Claim (PVC) for Splunk: Define a PVC to claim storage from the persistent volume.

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: splunk-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

```oc apply -f splunk-pvc.yaml```

Step 3: Deploy Splunk via Docker Image
Create a Deployment for Splunk: Create a YAML file that defines the Splunk deployment using the Docker image:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: splunk
  labels:
    app: splunk
spec:
  replicas: 1
  selector:
    matchLabels:
      app: splunk
  template:
    metadata:
      labels:
        app: splunk
    spec:
      containers:
      - name: splunk
        image: splunk/splunk:latest
        ports:
        - containerPort: 8000  # Web UI port
        - containerPort: 8089  # Management port
        - containerPort: 9997  # Forwarder receiving port
        - containerPort: 1514  # Syslog
        volumeMounts:
        - name: splunk-storage
          mountPath: /opt/splunk/var
        env:
        - name: SPLUNK_START_ARGS
          value: "--accept-license --answer-yes --no-prompt"
        - name: SPLUNK_PASSWORD
          value: "admin-password"  # Set the admin password
      volumes:
      - name: splunk-storage
        persistentVolumeClaim:
          claimName: splunk-pvc
```

```oc apply -f splunk-deployment.yaml```

Expose the Splunk Service: Expose the Splunk service so that it can be accessed from outside the OpenShift cluster.
Expose Splunk via an OpenShift service and route:

```
oc expose deployment/splunk --port=8000 --target-port=8000 --name=splunk-web
oc expose svc/splunk-web --hostname=splunk.example.com
```

This creates a route for the Splunk web interface.
Check the Deployment Status: Verify that Splunk is running:

```oc get pods -n splunk```

Step 4: Access Splunk
Access the Splunk Web Interface: Open your browser and go to the route URL you created:

Log in to Splunk: Use the admin credentials you set in the SPLUNK_PASSWORD environment variable:

Username: admin
Password: admin-password

Step 5: Check Logs: Monitor Splunk’s logs for any issues:

```oc logs deployment/splunk```

Scaling the Deployment: If you need to scale Splunk to handle more traffic, you can increase the number of replicas:

```oc scale deployment/splunk --replicas=3```
