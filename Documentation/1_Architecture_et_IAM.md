---
title: Projet Sécurité des Ressources - Déploiement Hospitalier
author: "[Staili Ilyes - Balikdjian Lucas - Massey raphael - Tekten Kaan / Groupe 4]"
date: 2026-02-17
tags:
  - Sécurité
  - Windows11
  - PowerShell
  - IAM
  - ACL
---

# DOSSIER TECHNIQUE : SÉCURISATION D'UNE INFRASTRUCTURE HOSPITALIÈRE (1/3)

> [!SUMMARY] Objectif du Document
> Ce document détaille l'architecture logicielle et la politique de gestion des identités (IAM) mises en place pour sécuriser l'environnement Windows 11 du centre hospitalier, conformément au cahier des charges "Projet sécurité des ressources".

---

## 1. Architecture de Déploiement (Infrastructure as Code)

Pour répondre à l'exigence d'automatisation et de maintenabilité, le déploiement ne repose pas sur un script monolithique, mais sur une architecture modulaire pilotée par des sources de vérité externes (fichiers JSON).

### 1.1. Modèle de Données (Fichiers JSON)
La topologie de l'hôpital est intégralement extraite du code métier. Trois fichiers de configuration dictent le comportement des scripts :

1. **`groupes.json`** : Définit la matrice RBAC (Role-Based Access Control) via les groupes locaux préfixés `G_` (ex: `G_Medecins`, `G_Direction`).
2. **`utilisateurs.json`** : Recense le personnel, son affectation, et son mot de passe initial de provisionnement.
3. **`structure.json`** : Cartographie l'arborescence des dossiers et définit les droits d'accès primaires (FullControl, Modify, etc.).

### 1.2. Architecture Modulaire des Scripts PowerShell
Afin de garantir un code lisible, auditable et respectant les standards de développement, les actions sont scindées en modules, pilotés par un chef d'orchestre :

* `0-Orchestrator.ps1` : Point d'entrée unique. Gère les paramètres d'exécution (installation vs. nettoyage) et appelle les sous-modules.
* `1-Identity.ps1` : Module IAM (Création des utilisateurs et groupes locaux).
* `2-Infrastructure.ps1` : Module Système de Fichiers (Création de l'arborescence et application des règles NTFS/ACL complexes).
* `3-Hardening.ps1` : Module de Durcissement (Restriction des exécutables critiques).

---

## 2. Gestion des Identités et Accès (IAM)

La phase de provisionnement des comptes locaux est gérée par le module `1-Identity.ps1`. Elle applique strictement les consignes de sécurité et les spécificités techniques requises.

### 2.1. Provisionnement des Groupes (RBAC)
Le script lit `groupes.json` et vérifie la non-existence préalable du groupe via un bloc conditionnel (`-not (Get-LocalGroup ...)`). Cela rend le script **idempotent** : il peut être relancé plusieurs fois sans générer d'erreurs d'interruption.

### 2.2. Création des Utilisateurs et "Subtilité" du Mot de Passe
Conformément aux exigences du document "Gestion des utilisateurs et des groupes", le provisionnement des mots de passe implique une contrainte technique majeure : **l'ingestion d'un mot de passe en texte clair depuis une source de données**.

> [!WARNING] Justification Technique de l'Exigence
> La cmdlet `New-LocalUser` exige un objet de type `SecureString` pour le paramètre `-Password`. Cependant, la donnée source (`defaultMotDePasse` dans le JSON) est une chaîne de caractères standard (String).

Pour satisfaire cette consigne, le script implémente la conversion forcée de la chaîne en clair vers un objet sécurisé avant la création du compte, via l'option `-AsPlainText -Force` :

```powershell
# Extraction du mot de passe en clair depuis le fichier de configuration JSON
$mdpEnClair = $u.defaultMotDePasse

# Conversion obligatoire pour satisfaire les exigences de New-LocalUser
$secPass = ConvertTo-SecureString $mdpEnClair -AsPlainText -Force

# Création de l'utilisateur avec un mot de passe qui n'expire pas (environnement de lab/démo)
New-LocalUser -Name $u.nom -FullName ($u.prenom + " " + $u.nom) -Password $secPass -PasswordNeverExpires
```

> [!SUCCESS] Conformité de l'Affichage
> Lors de l'exécution, le script affiche explicitement le mot de passe initial dans la console (ex: `[+] User créé : medecin1 (MDP: P@ssword1)`), permettant à l'administrateur ou à l'auditeur de valider la bonne prise en compte du fichier de configuration lors de la première connexion.

### 2.3. Affectation Dynamique
Les utilisateurs sont automatiquement liés à leur groupe métier. Le script concatène le préfixe `G_` avec le profil lu dans le JSON pour assurer une cohérence stricte entre l'identité et les droits (ex: Le profil "Medecins" est automatiquement affecté au groupe `G_Medecins`).