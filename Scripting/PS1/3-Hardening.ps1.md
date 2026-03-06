# Fichier : 3-Hardening.ps1
Write-Host "[*] PHASE 3 : HARDENING SYSTÈME (Ciblé et Sécurisé)" -ForegroundColor Cyan

# Cibles sensibles : Interpréteurs de commandes
$tools = @(
    "C:\Windows\System32\cmd.exe",
    "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
)

# La solution miracle : On cible UNIQUEMENT nos groupes métiers.
# Ainsi, l'Administrateur et le système restent libres de leurs mouvements.
$groupesMetiés = @("G_Direction", "G_RH", "G_Finance", "G_Medecins")

foreach ($tool in $tools) {
    if (Test-Path $tool) {
        try {
            $acl = Get-Acl $tool
            
            # On boucle sur chaque groupe métier pour lui appliquer la règle de Refus (Deny)
            foreach ($groupe in $groupesMetiés) {
                $ruleDeny = New-Object System.Security.AccessControl.FileSystemAccessRule($groupe, "ReadAndExecute", "Deny")
                $acl.AddAccessRule($ruleDeny)
            }
            
            Set-Acl -Path $tool -AclObject $acl -ErrorAction Stop
            Write-Host "   [+] Verrouillage activé (Ciblé) : $tool" -ForegroundColor Green
        } catch {
            Write-Host "   [!] Bypass simulé (Droits TrustedInstaller requis) : $tool" -ForegroundColor Yellow
        }
    }
}