pipeline {
  agent any

  environment {
    // change if you want a subfolder: 'C:\\inetpub\\wwwroot\\withoutscm'
    DEPLOY_DIR = 'C:\\inetpub\\wwwroot'
    // set this credential in Jenkins (username/token or username/password) and reference here
    GIT_CREDS  = 'github-pat'
    // git URL - update to your repo if different
    GIT_URL    = 'https://github.com/Aadiii1024/jenkins.git'
    GIT_BRANCH = 'master'
  }

  stages {

    stage('Checkout') {
      steps {
        echo "Cloning project from GitHub with credentialsId=${env.GIT_CREDS}"
        git branch: "${env.GIT_BRANCH}",
            url: "${env.GIT_URL}",
            credentialsId: "${env.GIT_CREDS}"
      }
    }

    stage('Build (none)') {
      steps {
        echo 'Static site — no build step.'
        // list files to help debugging
        powershell 'Write-Output \"Workspace: $env:WORKSPACE\"; Get-ChildItem -Recurse -Force -Depth 2 | Select-Object FullName,Length | Format-Table -AutoSize'
      }
    }

    stage('Deploy to IIS (mirror workspace)') {
      steps {
        powershell '''
Write-Output "=== Deploy: Start ==="
$src = $env:WORKSPACE
$dst = "''' + "${DEPLOY_DIR}" + '''"

Write-Output "Source: $src"
Write-Output "Destination: $dst"

# Create destination if missing
if (-not (Test-Path $dst)) {
    Write-Output "Creating destination folder $dst"
    New-Item -Path $dst -ItemType Directory -Force | Out-Null
} else {
    Write-Output "Destination already exists"
}

# Attempt robocopy mirror
$robocopy = Get-Command robocopy.exe -ErrorAction SilentlyContinue
if ($robocopy) {
    Write-Output "Using robocopy to mirror workspace -> destination"
    # /MIR mirrors; /XD excludes .git; /NFL/NJS reduce log noise
    $args = @($src, $dst, '/MIR', '/XD', '.git','.svn','.jenkins', '/NFL','/NDL','/NJH','/NJS','/NP')
    $proc = Start-Process -FilePath $robocopy.Path -ArgumentList $args -Wait -PassThru
    Write-Output "robocopy exit code: $($proc.ExitCode)"
    if ($proc.ExitCode -ge 8) {
        Write-Output "robocopy reported failure (exit >=8). Falling back to Copy-Item after cleaning target."
        Remove-Item -Path (Join-Path $dst '*') -Recurse -Force -ErrorAction SilentlyContinue
        Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force
    }
} else {
    Write-Output "robocopy not found — using Copy-Item fallback"
    Remove-Item -Path (Join-Path $dst '*') -Recurse -Force -ErrorAction SilentlyContinue
    Copy-Item -Path (Join-Path $src '*') -Destination $dst -Recurse -Force
}

Write-Output "Deploy copying complete."
Write-Output "=== Deploy: End ==="
'''
      }
    }

    stage('IIS ensure & configure') {
      steps {
        powershell '''
Write-Output "=== IIS Setup: Start ==="

$dst = "''' + "${DEPLOY_DIR}" + '''"

# Try to import WebAdministration
$iisAvailable = $false
try {
    Import-Module WebAdministration -ErrorAction Stop
    $iisAvailable = $true
    Write-Output "WebAdministration loaded: IIS management available."
} catch {
    Write-Output "WebAdministration module not available: $($_.Exception.Message)"
}

# If not available, try installing (best-effort; requires admin and appropriate OS)
if (-not $iisAvailable) {
    try {
        Write-Output "Attempting to install IIS (Install-WindowsFeature/Web-Server). This may fail on some SKUs..."
        Install-WindowsFeature -Name Web-Server -IncludeManagementTools -ErrorAction Stop
        Import-Module WebAdministration -ErrorAction Stop
        $iisAvailable = $true
        Write-Output "IIS installed and WebAdministration module loaded."
    } catch {
        Write-Output "IIS install attempt failed or not supported: $($_.Exception.Message)"
    }
}

if ($iisAvailable) {
    Write-Output "Configuring Default Web Site to use path: $dst"
    try {
        $site = Get-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
        if ($null -eq $site) {
            Write-Output "Default Web Site not found — creating it bound to port 80."
            New-Website -Name 'Default Web Site' -Port 80 -PhysicalPath $dst -Force
        } else {
            Write-Output "Default Web Site exists — updating physical path"
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

        Write-Output "Starting W3SVC and Default Web Site"
        Start-Service W3SVC -ErrorAction SilentlyContinue
        Start-Website -Name 'Default Web Site' -ErrorAction SilentlyContinue
    } catch {
        Write-Output "IIS config error: $($_.Exception.Message)"
    }
} else {
    Write-Output "IIS not available — skipped site creation/config. To host, install IIS on this machine."
}

# Set permissions so IIS can read files (IIS_IUSRS read/execute)
try {
    icacls $dst /grant 'IIS_IUSRS:(OI)(CI)RX' /T | Out-Null
    Write-Output "Granted IIS_IUSRS read/execute on $dst"
} catch {
    Write-Output "icacls failed: $($_.Exception.Message)"
}

Write-Output "=== IIS Setup: End ==="
'''
      }
    }

    stage('Verify site (localhost)') {
      steps {
        powershell '''
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
    Write-Output ("HTTP_ERROR: " + $_.Exception.Message)
    exit 3
}
'''
      }
    }

  } // stages

  post {
    success {
      echo "✅ Deployment finished. From this server open: http://localhost/index.html"
      echo "✅ From another machine open: http://<server-ip-or-hostname>/ (or /index.html)"
    }
    failure {
      echo "❌ Deployment or verification failed — check console output above for errors."
      echo "If IIS is not installed on this host, install IIS or change DEPLOY_DIR to point to a host that serves static files."
    }
  }
}

