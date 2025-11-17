pipeline {
  agent any

  environment {
    DEPLOY_DIR = 'C:\\inetpub\\wwwroot'
    GIT_CREDS  = 'github-pat'
    GIT_URL    = 'https://github.com/Aadiii1024/jenkins.git'
    GIT_BRANCH = 'master'
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Cloning project from GitHub with credentialsId=${env.GIT_CREDS}"
        git branch: "${env.GIT_BRANCH}", url: "${env.GIT_URL}", credentialsId: "${env.GIT_CREDS}"
      }
    }

    stage('Build (none)') {
      steps {
        echo 'Static site — no build step.'
        powershell 'Write-Output "Workspace: $env:WORKSPACE"; Get-ChildItem -Path $env:WORKSPACE -Recurse -Force -Depth 2 | Select-Object FullName,Length | Format-Table -AutoSize'
      }
    }

    stage('Deploy to IIS (mirror workspace)') {
      steps {
        powershell '''
$scriptPath = Join-Path $env:WORKSPACE "__tmp_deploy.ps1"
$here = @'
Write-Output "=== Deploy: Start ==="
$src = $env:WORKSPACE
$dst = $env:DEPLOY_DIR

Write-Output ("Source: {0}" -f $src)
Write-Output ("Destination: {0}" -f $dst)

if (-not (Test-Path -Path $dst)) {
    Write-Output ("Creating destination folder {0}" -f $dst)
    New-Item -Path $dst -ItemType Directory -Force | Out-Null
} else {
    Write-Output "Destination already exists"
}

$robocopy = Get-Command robocopy.exe -ErrorAction SilentlyContinue
if ($robocopy) {
    Write-Output "Using robocopy to mirror workspace -> destination"
    $args = @($src, $dst, '/MIR', '/XD', '.git','.svn','.jenkins', '/NFL','/NDL','/NJH','/NJS','/NP')
    $proc = Start-Process -FilePath $robocopy.Path -ArgumentList $args -Wait -PassThru
    Write-Output ("robocopy exit code: {0}" -f $proc.ExitCode)
    if ($proc.ExitCode -ge 8) {
        Write-Output "robocopy reported failure (exit >=8). Falling back to Copy-Item."
        Remove-Item -Path (Join-Path $dst '*') -Recurse -Force -ErrorAction SilentlyContinue
        Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force
    }
} else {
    Write-Output "robocopy not found - using Copy-Item fallback"
    Remove-Item -Path (Join-Path $dst '*') -Recurse -Force -ErrorAction SilentlyContinue
    Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force
}

Write-Output "Deploy copying complete."
Write-Output "=== Deploy: End ==="
'@

# write UTF8 without BOM to avoid encoding surprises
Set-Content -Path $scriptPath -Value $here -Encoding UTF8NoBOM
Write-Output ("Wrote deploy script to: {0}" -f $scriptPath)
Get-Content -Path $scriptPath -TotalCount 200 | ForEach-Object { Write-Output $_ }

# execute the temp ps1 in this runspace (call operator)
& $scriptPath
'''
      }
    }

    stage('IIS ensure & configure') {
      steps {
        powershell '''
$scriptPath = Join-Path $env:WORKSPACE "__tmp_iis_config.ps1"
$here = @'
Write-Output "=== IIS Setup: Start ==="
$dst = $env:DEPLOY_DIR

$iisAvailable = $false
try {
    Import-Module WebAdministration -ErrorAction Stop
    $iisAvailable = $true
    Write-Output "WebAdministration loaded: IIS management available."
} catch {
    Write-Output ("WebAdministration module not available: {0}" -f $_.Exception.Message)
}

if (-not $iisAvailable) {
    try {
        Write-Output "Attempting to install IIS (Install-WindowsFeature/Web-Server). This may fail on some SKUs..."
        Install-WindowsFeature -Name Web-Server -IncludeManagementTools -ErrorAction Stop
        Import-Module WebAdministration -ErrorAction Stop
        $iisAvailable = $true
        Write-Output "IIS installed and WebAdministration module loaded."
    } catch {
        Write-Output ("IIS install attempt failed or not supported: {0}" -f $_.Exception.Message)
    }
}

if ($iisAvailable) {
    Write-Output ("Configuring Default Web Site to use path: {0}" -f $dst)
    try {
        $site = Get-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
        if ($null -eq $site) {
            Write-Output "Default Web Site not found - creating it bound to port 80."
            New-Website -Name 'Default Web Site' -Port 80 -PhysicalPath $dst -Force
        } else {
            Write-Output "Default Web Site exists - updating physical path"
            Set-ItemProperty "IIS:\\Sites\\Default Web Site" -Name physicalPath -Value $dst -ErrorAction SilentlyContinue
        }

        Write-Output "Ensuring Default Document contains index.html"
        $dd = Get-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.'
        $hasIndex = $false
        foreach ($i in $dd.Collection) { if ($i.value -eq 'index.html') { $hasIndex = $true } }
        if (-not $hasIndex) {
            Add-WebConfigurationProperty -pspath 'MACHINE/WEBROOT/APPHOST' -filter 'system.webServer/defaultDocument/files' -name '.' -value @{value='index.html'}
            Write-Output "Added index.html to default documents"
        }

        Start-Service W3SVC -ErrorAction SilentlyContinue
        Start-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
    } catch {
        Write-Output ("IIS config error: {0}" -f $_.Exception.Message)
    }
} else {
    Write-Output "IIS not available - skipped site creation/config. To host, install IIS on this machine."
}

try {
    icacls $dst /grant 'IIS_IUSRS:(OI)(CI)RX' /T | Out-Null
    Write-Output ("Granted IIS_IUSRS read/execute on {0}" -f $dst)
} catch {
    Write-Output ("icacls failed: {0}" -f $_.Exception.Message)
}

Write-Output "=== IIS Setup: End ==="
'@

Set-Content -Path $scriptPath -Value $here -Encoding UTF8NoBOM
Write-Output ("Wrote IIS script to: {0}" -f $scriptPath)
Get-Content -Path $scriptPath -TotalCount 200 | ForEach-Object { Write-Output $_ }

& $scriptPath
'''
      }
    }

    stage('Verify site (localhost)') {
      steps {
        powershell '''
$scriptPath = Join-Path $env:WORKSPACE "__tmp_verify.ps1"
$here = @'
Write-Output "=== Verifying http://localhost/index.html ==="
try {
    $resp = Invoke-WebRequest -Uri "http://localhost/index.html" -UseBasicParsing -TimeoutSec 10
    Write-Output ("HTTP Status: " + $resp.StatusCode)
    if ($resp.RawContentLength -ne $null) { Write-Output ("Content-Length: " + $resp.RawContentLength) }
    Write-Output "--- Page Snippet ---"
    $snippet = if ($resp.Content.Length -gt 300) { $resp.Content.Substring(0,300) + "...(truncated)" } else { $resp.Content }
    Write-Output $snippet

    if ($resp.StatusCode -eq 200) {
        Write-Output "HTTP_OK"
        exit 0
    } else {
        Write-Output "HTTP_NON200"
        exit 2
    }
} catch {
    Write-Output ("HTTP_ERROR: {0}" -f $_.Exception.Message)
    exit 3
}
'@

Set-Content -Path $scriptPath -Value $here -Encoding UTF8NoBOM
Write-Output ("Wrote verify script to: {0}" -f $scriptPath)
Get-Content -Path $scriptPath -TotalCount 200 | ForEach-Object { Write-Output $_ }

& $scriptPath
'''
      }
    }
  } // stages

  post {
    success { echo "✅ Deployment finished. Open http://localhost/index.html" }
    failure { echo "❌ Deployment or verification failed — check console output above for errors." }
  }
}
