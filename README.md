# MediStock — Mini-projet 3

## Structure
- `manifests/` : namespace + RBAC + secret DB + quota + policies + NetworkPolicies + les 3 composants (DB/backend/frontend)
- `manifests-encryption-demo/` : StorageClass chiffree + Secret LUKS + PVC de demo pour le bloc "protection des donnees"

## Transferer sur la VM
Depuis votre machine locale, apres avoir telecharge et decompresse l'archive :
```bash
scp -r medistock/ zaikos@<IP_VM>:~/
```
(ou copiez le contenu via WinSCP/FileZilla/clé USB si plus simple)

## Ordre d'application (sur la VM)

```bash
cd ~/medistock/manifests

kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-serviceaccounts-rbac.yaml
kubectl apply -f 02-secret-db-credentials.yaml
kubectl apply -f 03-resourcequota.yaml
kubectl apply -f 04-admission-policies.yaml
kubectl apply -f 05-networkpolicies.yaml
kubectl apply -f 06-db-pvc.yaml
kubectl apply -f 07-db-deployment.yaml
kubectl apply -f 08-db-service.yaml
kubectl apply -f 09-backend-deployment.yaml
kubectl apply -f 10-backend-service.yaml
kubectl apply -f 11-frontend-deployment.yaml
kubectl apply -f 12-frontend-service.yaml
```

Ou en une seule commande (l'ordre numerique est respecte par kubectl) :
```bash
kubectl apply -f ~/medistock/manifests/
```

## Verification
```bash
kubectl get all -n medistock
kubectl get pvc -n medistock
kubectl get networkpolicy -n medistock
kubectl get validatingadmissionpolicies
kubectl get validatingadmissionpolicybindings
```

Tous les pods doivent passer `Running` (DB en premier car backend en depend indirectement,
mais Kubernetes retente automatiquement donc l'ordre d'apply n'est pas bloquant).

## Demo chiffrement (a appliquer separement, le jour de la presentation)
```bash
kubectl apply -f ~/medistock/manifests-encryption-demo/01-secret-longhorn-crypto.yaml
kubectl apply -f ~/medistock/manifests-encryption-demo/02-storageclass-longhorn-crypto.yaml
kubectl apply -f ~/medistock/manifests-encryption-demo/03-pvc-encrypted-demo.yaml
kubectl get pvc demo-encrypted-pvc -n medistock
```

## Tests de securite a faire pendant la demo

### 1. hostPath rejete
```bash
cat <<EOF2 | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
  namespace: medistock
spec:
  containers:
    - name: test
      image: busybox
      command: ["sleep", "3600"]
      volumeMounts:
        - name: host-vol
          mountPath: /host
  volumes:
    - name: host-vol
      hostPath:
        path: /
EOF2
```
Doit etre **refuse** par l'admission policy.

### 2. Frontend ne peut pas lire le secret DB
```bash
kubectl auth can-i get secret/medi-db-credentials \
  --as=system:serviceaccount:medistock:medi-frontend-sa -n medistock
```
Doit repondre `no`.

### 3. Backend peut lire SON secret (et seulement celui-la)
```bash
kubectl auth can-i get secret/medi-db-credentials \
  --as=system:serviceaccount:medistock:medi-backend-sa -n medistock
```
Doit repondre `yes`.

### 4. Communication frontend -> db bloquee (NetworkPolicy)
Depuis un pod frontend, tenter de joindre le port 5432 de medi-db doit timeout/echouer.

### 5. Quota de stockage
Creer un 6e PVC de 1Gi -> doit etre rejete par le ResourceQuota (max 5 PVC / 10Gi).
