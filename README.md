
Project — MySQL + phpMyAdmin on Kubernetes

Step 1 — Create Project Folder
bashmkdir ~/PhpMyAdmin
cd ~/PhpMyAdmin
Purpose: Keep all project files organized in one place.

Step 2 — Create Kubernetes Secret
Purpose: Never store passwords in plain YAML files. Secret encrypts and hides your password safely inside Kubernetes
vi secret.yml
Paste this:
yamlapiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
stringData:
  mysql-root-password: Admin@1234
  mysql-password: User@1234
Save → Ctrl+X → Y → Enter
Apply it:
bashkubectl apply -f secret.yml
Expected output:
secret/mysql-secret created

Step 3 — Create Persistent Volume
Purpose: MySQL stores all your database data here. Even if the pod crashes or restarts, your data is safe on disk.
bashnano pv.yml
Paste this:
yamlapiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/phpmyadmin-mysql
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
Save → Ctrl+X → Y → Enter
Apply it:
bashkubectl apply -f pv.yml
Expected output:
persistentvolume/mysql-pv created
persistentvolumeclaim/mysql-pvc created

Step 4 — Create MySQL Deployment + Service
Purpose: Runs the MySQL 8.0 database pod and creates an internal ClusterIP service so only phpMyAdmin can reach it — not the outside world.
bashnano mysql.yml
Paste this:
yamlapiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        - name: MYSQL_DATABASE
          value: myappdb
        - name: MYSQL_USER
          value: appuser
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  type: ClusterIP
Save → Ctrl+X → Y → Enter
Apply it:
bashkubectl apply -f mysql.yml
Expected output:
deployment.apps/mysql created
service/mysql-service created

Step 5 — Create phpMyAdmin Deployment + Service
Purpose: Runs the phpMyAdmin web UI and connects it to MySQL. NodePort makes it accessible from your browser.
bashnano phpmyadmin.yml
Paste this:
yamlapiVersion: apps/v1
kind: Deployment
metadata:
  name: phpmyadmin
  labels:
    app: phpmyadmin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: phpmyadmin
  template:
    metadata:
      labels:
        app: phpmyadmin
    spec:
      containers:
      - name: phpmyadmin
        image: phpmyadmin/phpmyadmin:latest
        env:
        - name: PMA_HOST
          value: mysql-service
        - name: PMA_PORT
          value: "3306"
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: mysql-root-password
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: phpmyadmin-service
spec:
  selector:
    app: phpmyadmin
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30095
  type: NodePort
Save → Ctrl+X → Y → Enter
Apply it:
bashkubectl apply -f phpmyadmin.yml
Expected output:
deployment.apps/phpmyadmin created
service/phpmyadmin-service created

Step 6 — Verify Everything is Running
bashkubectl get pods
kubectl get svc
kubectl get pvc
Expected output:
NAME                          READY   STATUS    RESTARTS   AGE
mysql-xxxxxxxxx-xxxxx         1/1     Running   0          2m
phpmyadmin-xxxxxxxxx-xxxxx    1/1     Running   0          1m

NAME                  TYPE        PORT(S)
mysql-service         ClusterIP   3306/TCP
phpmyadmin-service    NodePort    80:30095/TCP

NAME        STATUS   CAPACITY
mysql-pvc   Bound    2Gi

Step 7 — Port Forward to Browser
bashkubectl port-forward svc/phpmyadmin-service 9095:80 --address=0.0.0.0
Expected output:
Forwarding from 0.0.0.0:9095 -> 80
Forwarding from [::1]:9095 -> 80

Step 8 — Open in Windows Browser
http://localhost:9095
You will see the phpMyAdmin login page!
Login with:
Username: root
Password: Admin@1234

What you can do inside phpMyAdmin
Once logged in you can:
ActionHowCreate a new databaseClick "New" on left sidebarCreate tablesSelect database → Create tableRun SQL queriesClick "SQL" tabImport/Export dataClick "Import" or "Export" tabManage usersClick "User accounts" tabView all databasesLeft sidebar shows all databases

All your Kubernetes projects so far 🚀
ProjectPodsStatusNginx Web Server1 pod✅ RunningNode.js Todo App2 pods✅ RunningWeather App3 pods✅ RunningWordPress + MySQL2 pods✅ RunningMySQL + phpMyAdmin2 pods✅ New project

Run the steps and paste your kubectl get pods output here — let's get phpMyAdmin live in your browser! 🚀

![Uploading image.png…]()
