# Fichier : 4-Cleanup.ps1
# Rôle : Remettre la machine dans son état d'origine (Reverse Engineering du déploiement)

Write-Host "[!] DÉMARRAGE DU NETTOYAGE COMPLET DE L'ENVIRONNEMENT" -ForegroundColor Red

$ScriptPath = $PSScriptRoot
$pathU = "$ScriptPath\Json\utilisateurs.json"
$pathG = "$ScriptPath\Json\groupes.json"

# --- 1. SUPPRESSION DE L'ARBORESCENCE ---
$targetFolder = "C:\Hopital"
if (Test-Path $targetFolder) {
    Remove-Item -Path $targetFolder -Recurse -Force -ErrorAction SilentlyContinue
    Write-Host "   [-] Dossier et contenus supprimés : $targetFolder" -ForegroundColor Yellow
} else {
    Write-Host "   [i] Le dossier $targetFolder n'existe pas." -ForegroundColor DarkGray
}

# --- 2. SUPPRESSION DES UTILISATEURS ---
if (Test-Path $pathU) {
    $jsonU = Get-Content $pathU | ConvertFrom-Json
    foreach ($cat in $jsonU.profils) {
        foreach ($u in $cat.utilisateurs) {
            # On vérifie si l'utilisateur existe avant de le supprimer
            if (Get-LocalUser -Name $u.nom -ErrorAction SilentlyContinue) {
                Remove-LocalUser -Name $u.nom -ErrorAction SilentlyContinue
                Write-Host "   [-] Utilisateur supprimé : $($u.nom)" -ForegroundColor Yellow
            }
        }
    }
}

# --- 3. SUPPRESSION DES GROUPES ---
if (Test-Path $pathG) {
    $jsonG = Get-Content $pathG | ConvertFrom-Json
    foreach ($g in $jsonG.groupes) {
        # On vérifie si le groupe existe avant de le supprimer
        if (Get-LocalGroup -Name $g.nom -ErrorAction SilentlyContinue) {
            Remove-LocalGroup -Name $g.nom -ErrorAction SilentlyContinue
            Write-Host "   [-] Groupe supprimé : $($g.nom)" -ForegroundColor Yellow
        }
    }
}

Write-Host "[✅] ENVIRONNEMENT TOTALEMENT NETTOYÉ" -ForegroundColor Green