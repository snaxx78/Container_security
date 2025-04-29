# Rapport de TP : Sécurisation d'images Docker avec Cosign

## Introduction

Ce rapport présente les résultats du travail pratique sur la sécurisation d'images Docker à l'aide de Cosign, un outil permettant de signer et vérifier des conteneurs. L'objectif de ce TP était de comprendre le processus de signature cryptographique d'images Docker, leur publication dans un registry GitLab, puis la vérification de l'intégrité de ces images.

## Partie 1 : Génération des clés cryptographiques

La première étape consiste à installer Cosign et générer une paire de clés (publique/privée) permettant de signer les images Docker.

```bash
# Installation de cosign
curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o cosign
chmod +x cosign

# Génération de la paire de clés
./cosign generate-key-pair
```

Lors de l'exécution de la dernière commande, une passphrase est demandée pour sécuriser la clé privée. Cette étape a généré deux fichiers :
- `cosign.key` : la clé privée (à protéger)
- `cosign.pub` : la clé publique (à distribuer)

## Partie 2 : Build et signature d'une image Docker

J'ai ensuite créé une image Docker, l'ai publiée sur le registry GitLab, puis signée avec la clé privée précédemment générée.

```bash
# Construction et tag de l'image
docker build -t registry.gitlab.com/s-curit-des-containers/session-4/image:v1 .

# Connexion au registry GitLab
docker login registry.gitlab.com

# Envoi de l'image vers le registry
docker push registry.gitlab.com/s-curit-des-containers/session-4/image:v1

# Signature de l'image avec Cosign
.\cosign.exe sign --key cosign.key registry.gitlab.com/s-curit-des-containers/session-4/image:v1
```

Lors de l'exécution de la commande de signature, Cosign demande la passphrase définie précédemment pour déverrouiller la clé privée. La signature est ensuite stockée dans le registry GitLab sous forme d'une image spéciale contenant les informations cryptographiques.

## Partie 3 : Modification de l'image

Pour démontrer l'importance de la signature, j'ai créé une version modifiée de l'image originale.

```bash
# Création d'une modification (ajout d'un fichier vide "tampered")
docker run -it registry.gitlab.com/s-curit-des-containers/session-4/image:v1 /bin/sh

#Puis dans le conteneur:
touch /tampered.txt
exit

# Création d'une nouvelle image à partir du conteneur modifié
docker commit $(docker ps -lq) registry.gitlab.com/s-curit-des-containers/session-4/image:v2

# Envoi de l'image modifiée vers le registry
docker push registry.gitlab.com/s-curit-des-containers/session-4/image:v2
```

Cette nouvelle image v2 contient une modification non signée : un fichier "tampered" qui n'existait pas dans l'image originale.

## Partie 4 : Vérification des signatures

La dernière étape consiste à vérifier la signature des deux images pour s'assurer de l'intégrité de l'image originale et détecter la modification non autorisée de la seconde.

```bash
# Vérification de l'image originale
.\cosign.exe verify --key cosign.pub registry.gitlab.com/s-curit-des-containers/session-4/image:v1


# Vérification de l'image modifiée
.\cosign.exe verify --key cosign.pub registry.gitlab.com/s-curit-des-containers/session-4/image:v2
```

## Partie 5 : Résultats des commandes de vérification

### Vérification de l'image originale (v1)

La vérification de l'image v1 a réussi :

![image](https://github.com/user-attachments/assets/31de942f-3082-42c9-a02d-e7fb63698734)


Le message affiche les informations :
```
Verification for registry.gitlab.com/votre-projet/image:v1 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - Existence of the claims in the transparency log was verified offline
  - The signatures were verified against the specified public key
```

Cette validation confirme que l'image n'a pas été modifiée depuis sa signature, garantissant son intégrité.

### Vérification de l'image modifiée (v2)

La vérification de l'image v2 a échoué avec un message d'erreur :

![image](https://github.com/user-attachments/assets/9800cdfd-0cee-4b46-9435-b1ef2dd51b01)


Le message d'erreur indique :
```
Error: no matching signatures:
- no signatures found for image
```

Cet échec était attendu car l'image v2 n'a jamais été signée avec notre clé privée Cosign. Elle contient une modification (le fichier "tampered") qui n'était pas présente lors de la signature initiale.

