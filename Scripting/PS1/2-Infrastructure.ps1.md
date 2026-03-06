# Fichier : 2-Infrastructure.ps1
Write-Host "[*] PHASE 2 : DÉPLOIEMENT INFRASTRUCTURE & ACL" -ForegroundColor Cyan

$pathS = "$PSScriptRoot\Json\structure.json"
$jsonS = Get-Content $pathS | ConvertFrom-Json

foreach ($dossier in $jsonS.dossiers) {
    # Création du dossier physique
    if (-not (Test-Path $dossier.chemin)) {
        New-Item -ItemType Directory -Path $dossier.chemin | Out-Null
    }

    $acl = Get-Acl $dossier.chemin
    
    # Isolation : Rupture de l'héritage NTFS (Principe du moindre privilège)
    $acl.SetAccessRuleProtection($true, $false)

    # Règle de base (Propriétaire métier)
    $ruleBase = New-Object System.Security.AccessControl.FileSystemAccessRule($dossier.groupe, $dossier.droits, "ContainerInherit,ObjectInherit", "None", "Allow")
    $acl.AddAccessRule($ruleBase)

    # Règle Globale (L'IT a le FullControl partout)
    $ruleIT = New-Object System.Security.AccessControl.FileSystemAccessRule("G_IT", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
    $acl.AddAccessRule($ruleIT)

    # --- SCÉNARIOS DE CROISEMENTS COMPLEXES ---
    
    # Cas 1 : La Direction audite les RH (Lecture seule)
    if ($dossier.chemin -match "RH") {
        $ruleCross1 = New-Object System.Security.AccessControl.FileSystemAccessRule("G_Direction", "ReadAndExecute", "ContainerInherit,ObjectInherit", "None", "Allow")
        $acl.AddAccessRule($ruleCross1)
    }

    # Cas 2 : Les médecins déposent en Finance (Écriture en aveugle)
    if ($dossier.chemin -match "Finance") {
        $ruleCross2 = New-Object System.Security.AccessControl.FileSystemAccessRule("G_Medecins", "Write", "ContainerInherit,ObjectInherit", "None", "Allow")
        $acl.AddAccessRule($ruleCross2)
    }

    Set-Acl -Path $dossier.chemin -AclObject $acl
    Write-Host "   [+] Dossier sécurisé : $($dossier.chemin)" -ForegroundColor Green
}