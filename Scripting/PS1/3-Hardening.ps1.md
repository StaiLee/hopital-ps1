# Fichier : 3-Hardening.ps1
Write-Host "[*] PHASE 3 : HARDENING SYSTEME (Cible et Securise)" -ForegroundColor Cyan

$tools = @(
    "C:\Windows\System32\cmd.exe",
    "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe"
)

# Variable hyper simple, ZERO accent
$groupes = @("G_Direction", "G_RH", "G_Finance", "G_Medecins")

foreach ($tool in $tools) {
    if (Test-Path $tool) {
        try {
            $acl = Get-Acl $tool
            
            foreach ($g in $groupes) {
                $ruleDeny = New-Object System.Security.AccessControl.FileSystemAccessRule($g, "ReadAndExecute", "Deny")
                $acl.AddAccessRule($ruleDeny)
            }
            
            Set-Acl -Path $tool -AclObject $acl -ErrorAction Stop
            Write-Host "   [+] Verrouillage active (Cible) : $tool" -ForegroundColor Green
        } catch {
            Write-Host "   [!] Bypass simule (Droits TrustedInstaller requis) : $tool" -ForegroundColor Yellow
        }
    }
}