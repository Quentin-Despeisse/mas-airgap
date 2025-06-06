# Installation de MAS dans un environnement Airgap

Cette documentation a pour objectif de décrire comment installer MAS dans un cluster Openshift Airgap.

Elle a été testée dans les configurations suivantes :
- Openshift 4.16
- Maximo Application Suite v9.0.11
- IBM Maximo Operator Catalog v9-250501-amd64
- Podman v5.4.2
- MAS CLI v13.24.0

## Prérequis
- Un repository d'entreprise (Docker Registry, par exemple)
- Un cluster Openshift en v4.16
- Une station de travail ou Bastion équipé de [Docker](https://www.docker.com/products/docker-desktop/) ou [Podman](https://podman-desktop.io/) avec une connexion à internet et au repository d'entreprise
- Une [IBM Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary)
- Un [RedHat Pull Secret](https://console.redhat.com/openshift/install/pull-secret)
- Un fichier de licences Maximo Application Suite

## Copie des opérateurs
1. Définir les variables d'environnement :
```bash
export IBM_ENTITLEMENT_KEY=<Votre Entitlement Key>
export LOCAL_DIR=<Votre dossier de travail local>
export REGISTRY_HOST=<Nom d hôte de votre repository d entreprise>
export REGISTRY_PORT=<Port de votre repository d entreprise>
export REGISTRY_USERNAME=<Nom d utilisateur de votre repository d entreprise>
export REGISTRY_PASSWORD=<Mot de passe de votre repository d entreprise>
export REGISTRY_CA=<Nom du fichier contenant le certificat de votre repository d entreprise>
export REDHAT_SECRET=/mnt/registry/pull-secret.txt
export OCP_VERSION=4.16
export CLI_VERSION=13.24.0-amd64
export CATALOG_VERSION=v9-250501-amd64
export MAS_CHANNEL=9.0.x
export SSL_CERT_FILE=$LOCAL_DIR/$REGISTRY_CA
```
2. Déposer le pull secret de RedHat et le certificat du Container Registry dans le dossier `$LOCAL_DIR`
3. Copier l'image du CLI en local :
```bash
oc image mirror --dir $LOCAL_DIR/cli quay.io/ibmmas/cli:$CLI_VERSION file://ibmmas/cli:$CLI_VERSION
```
4. Télécharger l'image du CLI sur le repository d'entreprise :
```bash
podman login $REGISTRY_HOST:$REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD --cert-dir $LOCAL_DIR
oc image mirror --dir $LOCAL_DIR/cli file://ibmmas/cli:$CLI_VERSION $REGISTRY_HOST:$REGISTRY_PORT/ibmmas/cli:$CLI_VERSION
```
5. Copier les images RedHat requises pour l'installation de MAS :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/registry quay.io/ibmmas/cli:$CLI_VERSION mas mirror-redhat-images --mode direct --dir /mnt/registry/redhat -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD --pullsecret $REDHAT_SECRET --mirror-operators --release $OCP_VERSION --no-confirm
```
6. Copier les images de MAS Core :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/registry quay.io/ibmmas/cli:$CLI_VERSION mas mirror-images -m direct -d /mnt/registry/core -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD -c $CATALOG_VERSION -C $MAS_CHANNEL --mirror-catalog --mirror-core --ibm-entitlement $IBM_ENTITLEMENT_KEY
```
7. Copier les images MAS Manage :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/registry quay.io/ibmmas/cli:$CLI_VERSION mas mirror-images -m direct -d /mnt/registry/apps -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD -c $CATALOG_VERSION -C $MAS_CHANNEL --mirror-manage --ibm-entitlement $IBM_ENTITLEMENT_KEY
```
8. Copier les images des dépendances :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/registry quay.io/ibmmas/cli:$CLI_VERSION mas mirror-images -m direct -d /mnt/registry/others -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD -c $CATALOG_VERSION -C $MAS_CHANNEL --mirror-mongo --mirror-tsm --mirror-sls --ibm-entitlement $IBM_ENTITLEMENT_KEY 
```
9. Vérifier que l'intégralité des images a bien été copiée :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/registry quay.io/ibmmas/cli:$CLI_VERSION mas mirror-images -m direct -d /mnt/registry/others -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD -c $CATALOG_VERSION -C $MAS_CHANNEL --mirror-catalog --mirror-manage  --mirror-mongo --mirror-tsm --mirror-sls --ibm-entitlement $IBM_ENTITLEMENT_KEY
```

## Préparation du Cluster
1. Déposer le fichier de licences dans le dossier `$LOCAL_DIR`
2. Lancer le CLI d'installation MAS :
```bash
podman run -ti --rm --platform linux/amd64 -v $LOCAL_DIR:/mnt/home $REGISTRY_HOST:$REGISTRY_PORT/ibmmas/cli:$CLI_VERSION
```
3. Se connecter au cluster Openshift depuis le CLI en copiant la commande de connexion depuis la console :
![login_cmd.png](img/login_cmd.png)
4. Copier le certificat du repository d'entreprise dans le dossier `$LOCAL_DIR`
5. Rejouer les commandes pour définir les variables d'environnement
6. Configurer les catalogues sur Openshift :
```bash
mas configure-airgap --setup-redhat-catalogs -H $REGISTRY_HOST -P $REGISTRY_PORT -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD --ca-file /mnt/home/$REGISTRY_CA --no-confirm
```
6. Créer la ImageTagMirrorSet en cliquant sur le + en haut à droite de la console Openshift :

```yml
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: image-policy-quay-tag
spec:
  imageTagMirrors:
    - mirrors:
        - '$REGISTRY_HOST:$REGISTRY_PORT/ibmmas'
      source: quay.io/ibmmas
```

## Installation de MAS
1. Toujours dans le conteneur du CLI, déposer le fichier de licences MAS et le certificat de la BDD dans le dossier `$LOCAL_DIR`
2. Paramétrer le fichier `env.sh` fourni et l'exécuter 
3. Se placer dans le dossier `$LOCAL_DIR` et créer le dossier `mascfg`
```bash
mkdir mascfg
```
3. Lancer l'installation de MAS Core (Environ 2h):
```bash
ansible-playbook ibm.mas_devops.oneclick_core
```
4. Lancer l'installation de MAS Manage (Environ 4h):
```bash
ansible-playbook ibm.mas_devops.oneclick_add_manage
```