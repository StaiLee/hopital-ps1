# Fichier : 3-Hardening.ps1
Write-Host "[*] PHASE 3 : HARDENING SYSTÈME" -ForegroundColor Cyan

# Cibles sensibles : Interpréteurs de commandes
$tools = @(
    "C:\Windows\System32\cmd.exe",
    "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
)

# Note : Nécessite une appropriation préalable (Takeown) en condition réelle
foreach ($tool in $tools) {
    if (Test-Path $tool) {
        try {
            $acl = Get-Acl $tool
            # Règle de REFUS explicite (Prend le pas sur l'autorisation)
            $ruleDeny = New-Object System.Security.AccessControl.FileSystemAccessRule("Utilisateurs", "ReadAndExecute", "Deny")
            $acl.AddAccessRule($ruleDeny)
            Set-Acl -Path $tool -AclObject $acl -ErrorAction Stop
            Write-Host "   [+] Verrouillage activé : $tool" -ForegroundColor Green
        } catch {
            Write-Host "   [!] Bypass simulé (Droits TrustedInstaller requis) : $tool" -ForegroundColor Yellow
        }
    }
}