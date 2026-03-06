---
title: Monitoring Temps Réel (SOC) et Conclusion
author: "[Staili Ilyes - Balikdjian Lucas - Massey raphael - Tekten Kaan / Groupe 4]"
date: 2026-02-17
tags:
  - Sécurité
  - SOC
  - Golang
  - Audit
  - Observabilité
---

# DOSSIER TECHNIQUE : SÉCURISATION D'UNE INFRASTRUCTURE HOSPITALIÈRE (3/3)

> [!SUMMARY] Objectif du Document
> Ce document présente la surcouche de sécurité avancée du projet : l'implémentation d'un mini-SOC (Security Operations Center) en temps réel pour l'audit des ressources, ainsi que le bilan final du déploiement.

---

## 6. Audit et Observabilité (Dashboard Temps Réel)

Appliquer des règles de sécurité NTFS et durcir le système est une première étape de protection (Prévention). La seconde étape indispensable en cybersécurité est la **Détection**. 
Pour valider l'efficacité de notre architecture, nous avons développé une sonde d'audit en temps réel.

### 6.1. Activation de la Stratégie d'Audit Windows (AuditPol)
Afin de rendre le noyau Windows "bavard" sur les actions critiques, la stratégie d'audit locale a été modifiée en profondeur. Le système traque désormais les succès et les échecs sur trois axes majeurs :

```powershell
auditpol /set /category:"Gestion des comptes" /success:enable /failure:enable
auditpol /set /category:"Accès aux objets" /success:enable /failure:enable
auditpol /set /category:"Ouverture/Fermeture de session" /success:enable /failure:enable
```
* **Gestion des comptes :** Surveille les élévations de privilèges et la création/suppression de personnel.
* **Accès aux objets :** Permet de prouver que les règles NTFS (ACL) bloquent bien les accès non autorisés (EventID 4663 / 4624).

### 6.2. Le Moteur d'Analyse (Golang TUI)
Plutôt que d'obliger l'administrateur à fouiller manuellement dans l'Observateur d'événements (Event Viewer), nous avons développé une interface TUI (Terminal User Interface) en **Golang**.

> [!INFO] Architecture du Dashboard
> * **Backend (Go) :** Utilise des goroutines pour interroger le système de manière asynchrone sans bloquer l'interface.
> * **Frontend (tview/tcell) :** Affiche une matrice de sécurité divisée en trois panneaux : Base de données Utilisateurs (Live), Groupes Locaux (Live), et le Flux d'Audit de Sécurité.
> * **Connecteur :** Exécute des appels PowerShell optimisés (`Get-WinEvent`) et parse les retours JSON ou XML pour extraire la "Cible" de l'action de manière chirurgicale.

### 6.3. Événements Sécuritaires Traqués
Le système filtre le bruit pour se concentrer sur les indicateurs de compromission (IoC) ou de modification d'infrastructure :
* **Event 4720 :** Création d'un compte (Provisioning).
* **Event 4726 :** Suppression d'un compte (Offboarding).
* **Event 4732 / 4733 :** Modification des groupes locaux (Changement de droits RBAC).
* **Event 4663 :** Tentative d'accès à un fichier (Prouve le fonctionnement des ACLs sur l'arborescence médicale).

> [!SUCCESS] Résultat Opérationnel
> Lorsqu'un déploiement via `0-Orchestrator.ps1` est lancé, le Dashboard s'illumine en temps réel, catégorisant visuellement chaque métier (Médecins, IT, RH) et affichant instantanément la création des arborescences. C'est la preuve visuelle et technique que le code PowerShell impacte bien le système.

---

## 7. Bilan et Conclusion du Projet

L'infrastructure hospitalière est désormais déployée et sécurisée conformément aux exigences strictes du cahier des charges.

### 7.1. Objectifs Atteints
1. **Séparation des privilèges :** Le modèle RBAC isole parfaitement les entités (Direction, RH, IT, Médical).
2. **Infrastructure as Code :** Le déploiement est 100% automatisé, reproductible, et piloté par des données agnostiques (fichiers JSON).
3. **Sécurité en Profondeur :** Croisement des permissions (NTFS) + Verrouillage des exécutables (Hardening) + Traçabilité (Audit Go).

### 7.2. Pistes d'Amélioration (V2)
Si ce projet devait être porté en production réelle (hors environnement de laboratoire local), les évolutions suivantes seraient requises :
* **Active Directory :** Migration des comptes locaux (`New-LocalUser`) vers un annuaire centralisé (`New-ADUser`) pour une gestion multi-postes.
* **GPO (Group Policy Object) :** Déploiement du Hardening (blocage de `cmd.exe`) via des stratégies de domaine plutôt que par manipulation d'ACL locale.
* **Centralisation des Logs :** Envoi des événements de sécurité vers un SIEM externe (ex: Splunk, ELK) pour éviter la perte de traces en cas de compromission de la machine locale.

---
*Fin du document technique.*