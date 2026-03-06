---
title: Sécurité NTFS, Hardening et Cycle de Vie
author: "[Staili Ilyes - Balikdjian Lucas - Massey raphael - Tekten Kaan / Groupe 4]"
date: 2026-02-17
tags:
  - Sécurité
  - NTFS
  - ACL
  - Hardening
  - Nettoyage
---

# DOSSIER TECHNIQUE : SÉCURISATION D'UNE INFRASTRUCTURE HOSPITALIÈRE (2/3)

> [!SUMMARY] Objectif du Document
> Ce document détaille l'implémentation de la politique de contrôle d'accès (ACL NTFS), la résolution des scénarios d'accès complexes, les mesures de durcissement du système d'exploitation, ainsi que la gestion du cycle de vie de l'environnement (Nettoyage).

---

## 3. Matrice de Sécurité et Contrôle d'Accès (NTFS)

La sécurisation des ressources hospitalières repose sur le module `2-Infrastructure.ps1`. Le principe fondamental appliqué ici est le **Zero Trust (Moindre Privilège)** couplé à une isolation stricte des répertoires.

### 3.1. Rupture d'Héritage (Isolation)
Par défaut, Windows propage les droits du répertoire parent aux sous-dossiers. Pour garantir qu'aucun utilisateur standard n'accède à un dossier métier par héritage involontaire, le script brise cette chaîne de manière chirurgicale avant toute affectation de droits :

```powershell
# Instanciation de l'objet ACL
$acl = Get-Acl $fullPath
# Désactivation de l'héritage (IsProtected = $true, PreserveInheritance = $true)
$acl.SetAccessRuleProtection($true, $true)
```

### 3.2. Implémentation des 3 Cas Complexes (Croisements)
Conformément au cahier des charges exigeant au minimum trois scénarios de croisement complexe des permissions, l'architecture implémente les règles spécifiques suivantes :

> [!DANGER] Scénario Complexe 1 : Audit RH par la Direction
> **Besoin :** Le groupe `G_Direction` doit pouvoir consulter les dossiers du personnel sans pouvoir les altérer.
> **Solution :** Injection d'une règle `ReadAndExecute` explicite sur le dossier `RH` uniquement pour la Direction.
> ```powershell
> $ruleCross = New-Object System.Security.AccessControl.FileSystemAccessRule("G_Direction", "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")
> $acl.AddAccessRule($ruleCross)
> ```

> [!DANGER] Scénario Complexe 2 : Dépôt de factures (Drop-Box)
> **Besoin :** Le groupe `G_Medecins` doit pouvoir déposer des documents comptables dans le dossier `Finance`, mais ne doit ni lire les fichiers existants, ni modifier les dossiers.
> **Solution :** Création d'une règle d'écriture "aveugle" (`Write`) sans droits de lecture.
> ```powershell
> $ruleCross = New-Object System.Security.AccessControl.FileSystemAccessRule("G_Medecins", "Write", "ContainerInherit,ObjectInherit", "None", "Allow")
> ```

> [!DANGER] Scénario Complexe 3 : Super-Administration IT
> **Besoin :** Le service informatique (`G_IT`) doit pouvoir maintenir l'ensemble de l'arborescence, quel que soit le propriétaire du dossier.
> **Solution :** Application globale et systématique d'une règle `FullControl` pour `G_IT` sur chaque itération de la boucle de création des dossiers.

---

## 4. Durcissement du Système (Hardening)

Pour éviter qu'un utilisateur métier (ex: un infirmier ou un médecin) ne tente de contourner les restrictions via des lignes de commande, le module `3-Hardening.ps1` verrouille les binaires d'administration de l'OS.

### 4.1. Restriction des Interpréteurs de Commandes
Le script cible `cmd.exe` et `powershell.exe`. Une règle de refus explicite (`Deny`) est appliquée au groupe local `Utilisateurs` (qui englobe tous les comptes standards créés, mais n'affecte pas les Administrateurs locaux).

```powershell
$tools = @("C:\Windows\System32\cmd.exe", "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe")

foreach ($tool in $tools) {
    $acl = Get-Acl $tool
    # Création de la règle de REFUS d'exécution
    $ruleDeny = New-Object System.Security.AccessControl.FileSystemAccessRule("Utilisateurs", "ReadAndExecute", "Deny")
    $acl.AddAccessRule($ruleDeny)
    Set-Acl -Path $tool -AclObject $acl
}
```
> [!NOTE] Précédence des ACL
> Dans l'architecture Windows NTFS, une règle `Deny` prend toujours le pas sur une règle `Allow`. Ainsi, même si l'utilisateur possède des droits de lecture natifs, l'exécution est bloquée au niveau du noyau.

---

## 5. Gestion du Cycle de Vie et Mode "Nettoyage"

Le sujet impose de pouvoir remettre l'environnement dans son état initial. Le module `0-Orchestrator.ps1` gère cette fonction via un système de paramètres d'exécution (`switch`).

### 5.1. Le paramètre `-ClearAll`
L'exécution de la commande `.\0-Orchestrator.ps1 -ClearAll` déclenche la procédure de dé-provisionnement.
Le script procède à un "Reverse Engineering" de l'installation :
1. **Dossiers :** Suppression récursive de l'arborescence (`Remove-Item "C:\Hopital" -Recurse -Force`).
2. **Utilisateurs :** Reparcourt le fichier `utilisateurs.json` pour cibler et exécuter `Remove-LocalUser` uniquement sur les comptes créés par le projet (laissant les comptes vitaux de Windows intacts).
3. **Groupes :** Répète la même logique de ciblage avec `Remove-LocalGroup` via `groupes.json`.

Cette approche garantit un environnement de laboratoire propre et réutilisable pour de futures itérations ou audits.