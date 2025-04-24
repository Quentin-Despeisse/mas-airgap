# Installation de MAS dans un environnement Airgap

Cette documentation a pour objectif de décrire comment installer MAS dans un cluster Openshift Airgap.

Elle a été testée dans les configurations suivantes :
- Openshift 4.16
- Maximo Application Suite v9.0.10
- IBM Maximo Operator Catalog v9-250306-amd64
- Podman v5.4.2
- MAS CLI v13.15.0

## Prérequis
- Un repository d'entreprise (Docker Registry, par exemple)
- Un cluster Openshift en v4.16
- Une station de travail ou Bastion équipé de [Docker](https://www.docker.com/products/docker-desktop/) ou [Podman](https://podman-desktop.io/) avec une connexion à internet et au repository d'entreprise
- Une [IBM Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary)
- Un [RedHat Pull Secret](https://console.redhat.com/openshift/install/pull-secret)

## Copie des opérateurs
1. Définir les variables d'environnement :
```bash
export IBM_ENTITLEMENT_KEY=<Votre Entitlement Key>
export LOCAL_DIR=<Votre dossier de travail local>
export REGISTRY_HOST=<Nom d hôte de votre repository d entreprise>
export REGISTRY_PORT=<Port de votre repository d entreprise>
export REGISTRY_USERNAME=<Nom d utilisateur de votre repository d entreprise>
export REGISTRY_PASSWORD=<Mot de passe de votre repository d entreprise>
export REDHAT_SECRET=<Chemin vers le pull secret RedHat>
```
2. Copier l'image du CLI en local :
```bash
oc image mirror --dir $LOCAL_DIR/cli quay.io/ibmmas/cli:13.15.0 file://ibmmas/cli:13.15.0
```