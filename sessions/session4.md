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
docker build -t registry.gitlab.com/votre-projet/image:v1 .

# Connexion au registry GitLab
docker login registry.gitlab.com

# Envoi de l'image vers le registry
docker push registry.gitlab.com/votre-projet/image:v1

# Signature de l'image avec Cosign
./cosign sign --key cosign.key registry.gitlab.com/votre-projet/image:v1
```

Lors de l'exécution de la commande de signature, Cosign demande la passphrase définie précédemment pour déverrouiller la clé privée. La signature est ensuite stockée dans le registry GitLab sous forme d'une image spéciale contenant les informations cryptographiques.

## Partie 3 : Modification de l'image

Pour démontrer l'importance de la signature, j'ai créé une version modifiée de l'image originale.

```bash
# Création d'une modification (ajout d'un fichier vide "tampered")
docker run -it --rm registry.gitlab.com/votre-projet/image:v1 touch /tampered

# Création d'une nouvelle image à partir du conteneur modifié
docker commit $(docker ps -lq) registry.gitlab.com/votre-projet/image:v2

# Envoi de l'image modifiée vers le registry
docker push registry.gitlab.com/votre-projet/image:v2
```

Cette nouvelle image v2 contient une modification non signée : un fichier "tampered" qui n'existait pas dans l'image originale.

## Partie 4 : Vérification des signatures

La dernière étape consiste à vérifier la signature des deux images pour s'assurer de l'intégrité de l'image originale et détecter la modification non autorisée de la seconde.

```bash
# Vérification de l'image originale
./cosign verify --key cosign.pub registry.gitlab.com/votre-projet/image:v1

# Vérification de l'image modifiée
./cosign verify --key cosign.pub registry.gitlab.com/votre-projet/image:v2
```

## Partie 5 : Résultats des commandes de vérification

### Vérification de l'image originale (v1)

La vérification de l'image v1 a réussi avec un résultat similaire à :

[Insérer capture d'écran ici - Image v1 verification]

Le message affiche typiquement des informations comme :
```
Verification for registry.gitlab.com/votre-projet/image:v1 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
```

Cette validation confirme que l'image n'a pas été modifiée depuis sa signature, garantissant son intégrité.

### Vérification de l'image modifiée (v2)

La vérification de l'image v2 a échoué avec un message d'erreur similaire à :

[Insérer capture d'écran ici - Image v2 verification failure]

Le message d'erreur indique généralement :
```
Error: no matching signatures:
- no signatures found for image
```

Cet échec était attendu car l'image v2 n'a jamais été signée avec notre clé privée Cosign. Elle contient une modification (le fichier "tampered") qui n'était pas présente lors de la signature initiale.

