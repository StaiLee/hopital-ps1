# Fichier : 1-Identity.ps1
Write-Host "[*] PHASE 1 : GESTION DES IDENTITES (IAM)" -ForegroundColor Cyan

$pathG = "$PSScriptRoot\Json\groupes.json"
$pathU = "$PSScriptRoot\Json\utilisateurs.json"

# Creation des groupes locaux (Idempotent)
$jsonG = Get-Content $pathG | ConvertFrom-Json
foreach ($g in $jsonG.groupes) {
    if (-not (Get-LocalGroup -Name $g.nom -ErrorAction SilentlyContinue)) {
        New-LocalGroup -Name $g.nom -Description $g.description | Out-Null
        Write-Host "   [+] Groupe cree : $($g.nom)" -ForegroundColor Green
    }
}

# Creation des utilisateurs et affectation RBAC
$jsonU = Get-Content $pathU | ConvertFrom-Json
foreach ($cat in $jsonU.profils) {
    $targetGroup = "G_" + $cat.profil
    foreach ($u in $cat.utilisateurs) {
        # Conversion du MDP clair en SecureString (Exigence technique)
        $secPass = ConvertTo-SecureString $u.defaultMotDePasse -AsPlainText -Force
        
        if (-not (Get-LocalUser -Name $u.nom -ErrorAction SilentlyContinue)) {
            New-LocalUser -Name $u.nom -FullName ($u.prenom + " " + $u.nom) -Password $secPass -PasswordNeverExpires | Out-Null
            Write-Host "   [+] User cree : $($u.nom) (MDP: $($u.defaultMotDePasse))" -ForegroundColor Green
            
            # 1. Ajout au groupe metier (Ex: G_Medecins)
            Add-LocalGroupMember -Group $targetGroup -Member $u.nom
            Write-Host "       -> Rattache au profil : $targetGroup" -ForegroundColor Cyan
            
            # 2. FIX VISIBILITE : Ajout au groupe standard de Windows pour l'ecran d'accueil
            Add-LocalGroupMember -Group "Utilisateurs" -Member $u.nom -ErrorAction SilentlyContinue
        }
    }
}