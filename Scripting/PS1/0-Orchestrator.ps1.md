# Fichier : 0-Orchestrator.ps1
param (
    [switch]$ClearAll
)

$ScriptPath = $PSScriptRoot

if ($ClearAll) {
    Write-Host "[!] MODE NETTOYAGE ACTIVÉ" -ForegroundColor Red
    
    # 1. Suppression des dossiers
    if (Test-Path "C:\Hopital") { Remove-Item "C:\Hopital" -Recurse -Force }
    
    # 2. Suppression dynamique des utilisateurs créés par le projet
    $jsonU = Get-Content "$ScriptPath\Json\utilisateurs.json" | ConvertFrom-Json
    foreach ($cat in $jsonU.profils) {
        foreach ($u in $cat.utilisateurs) {
            Remove-LocalUser -Name $u.nom -ErrorAction SilentlyContinue
        }
    }
    
    # 3. Suppression dynamique des groupes
    $jsonG = Get-Content "$ScriptPath\Json\groupes.json" | ConvertFrom-Json
    foreach ($g in $jsonG.groupes) {
        Remove-LocalGroup -Name $g.nom -ErrorAction SilentlyContinue
    }
    
    Write-Host "[+] Nettoyage terminé." -ForegroundColor Green
    exit
}

Write-Host "🚀 DÉMARRAGE DU DÉPLOIEMENT" -ForegroundColor Magenta
& "$ScriptPath\1-Identity.ps1"
& "$ScriptPath\2-Infrastructure.ps1"
& "$ScriptPath\3-Hardening.ps1"
Write-Host "✅ DÉPLOIEMENT TERMINÉ" -ForegroundColor Magenta