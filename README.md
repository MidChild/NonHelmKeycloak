# Créer un cluster Kind pour le développement
kind create cluster --name keycloak-dev

# Vérifier que le cluster est opérationnel
kubectl cluster-info --context kind-keycloak-dev

# Vérifier les nodes
kubectl get nodes


# Créer le namespace de développement
kubectl create namespace dev

# Vérifier la création
kubectl get namespaces


# Appliquer le PersistentVolumeClaim pour PostgreSQL
kubectl apply -f pvc.yaml

# Vérifier que le PVC est créé
kubectl get pvc -n dev


# Appliquer la configuration PostgreSQL (ConfigMap)
kubectl apply -f postgresconfigmap.yaml

# Créer le déploiement PostgreSQL (si non existant)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: postgres-secret
              key: POSTGRES_DB
        - name: POSTGRES_USER
          valueFrom:
            configMapKeyRef:
              name: postgres-secret
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            configMapKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        - name: PGDATA
          value: "/var/lib/postgresql/data/pgdata"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
EOF

# Déployer le service PostgreSQL
kubectl apply -f pgservice.yaml

# Attendre que PostgreSQL soit prêt (timeout 5 minutes)
kubectl wait --for=condition=ready pod -l app=postgres -n dev --timeout=300s

# Vérifier les logs PostgreSQL
kubectl logs -n dev -l app=postgres


# Corriger le déploiement Keycloak avec la bonne configuration (si incorrecte)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak-development
  namespace: dev
spec:
  selector:
    matchLabels:
      app: keycloak-development
  replicas: 1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: keycloak-development
    spec:
      containers:
        - name: keycloak-development
          image: quay.io/keycloak/keycloak:26.4
          args: ["start-dev"]
          imagePullPolicy: Always
          env:
            - name: KEYCLOAK_ADMIN
              value: "admin"
            - name: KEYCLOAK_ADMIN_PASSWORD
              value: "admin"
            - name: KC_DB
              value: "postgres"
            - name: KC_DB_URL
              value: "jdbc:postgresql://postgres:5432/appdb"
            - name: KC_DB_USERNAME
              value: "appuser"
            - name: KC_DB_PASSWORD
              value: "strongpasswordapp"
            - name: KC_HTTP_ENABLED
              value: "true"
            - name: KC_HOSTNAME_STRICT
              value: "false"
            - name: KC_HOSTNAME_STRICT_HTTPS
              value: "false"
          ports:
            - containerPort: 8080
EOF

# Déployer le service Keycloak
kubectl apply -f service.yaml

# Attendre que Keycloak soit prêt (timeout 5 minutes)
kubectl wait --for=condition=ready pod -l app=keycloak-development -n dev --timeout=300s

# Vérifier les logs Keycloak
kubectl logs -n dev -l app=keycloak-development


# Vérifier que tous les pods sont en cours d'exécution
kubectl get pods -n dev

# Vérifier les services
kubectl get services -n dev

# Vérifier les logs pour s'assurer qu'il n'y a pas d'erreurs
kubectl logs -n dev -l app=postgres | tail -10
kubectl logs -n dev -l app=keycloak-development | tail -10


# Port-forward pour accéder à Keycloak localement
kubectl port-forward -n dev svc/keycloak-development 8080:53582

Accès depuis http://localhost:8080