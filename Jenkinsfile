// Jenkins obtiene este archivo y la app desde GitHub para simular una entrega.
// RUTA_KIT apunta al laboratorio local; sus secretos nunca se copian a GitHub.
pipeline {
    agent any
    options { skipDefaultCheckout() }

    stages {
        stage('1. GitHub y preparar') {
            steps {
                checkout scm
                powershell '''
                    $ErrorActionPreference = 'Stop'
                    git -C $env:WORKSPACE log -1 --format='%h %s'
                    if ($LASTEXITCODE -ne 0) { throw 'No se pudo leer el commit de Git' }
                    if (-not (Test-Path -LiteralPath $env:RUTA_KIT -PathType Container)) {
                        throw 'RUTA_KIT no apunta a una carpeta. Revisa el parámetro del trabajo.'
                    }
                    if (-not (Test-Path -LiteralPath (Join-Path $env:WORKSPACE 'compose.yaml') -PathType Leaf)) {
                        throw 'Jenkins no descargó el kit desde GitHub'
                    }
                    robocopy $env:WORKSPACE $env:RUTA_KIT /E /XD .git secrets __pycache__ /XF *.pyc /NFL /NDL /NJH /NJS /NC /NS | Out-Null
                    if ($LASTEXITCODE -ge 8) { throw 'No se pudo copiar el código de GitHub al kit local' }
                    Set-Location -LiteralPath $env:RUTA_KIT
                    if (-not (Test-Path -LiteralPath 'compose.yaml' -PathType Leaf)) {
                        throw 'No se encontró compose.yaml en RUTA_KIT'
                    }
                    docker info *> $null
                    if ($LASTEXITCODE -ne 0) { throw 'Jenkins no puede usar Docker Desktop' }
                    & (Join-Path (Get-Location).Path 'preparar-secretos.ps1')
                    docker compose config --quiet
                    if ($LASTEXITCODE -ne 0) { throw 'La configuración de Docker Compose no es válida' }
                '''
            }
        }

        stage('2. Construir app') {
            steps {
                powershell '''
                    $ErrorActionPreference = 'Stop'
                    Set-Location -LiteralPath $env:RUTA_KIT
                    docker compose build flask_service
                    if ($LASTEXITCODE -ne 0) { throw 'La construcción de la app falló' }
                '''
            }
        }

        stage('3. Desplegar local') {
            steps {
                powershell '''
                    $ErrorActionPreference = 'Stop'
                    Set-Location -LiteralPath $env:RUTA_KIT
                    docker compose up -d --no-build
                    if ($LASTEXITCODE -ne 0) { throw 'El despliegue local falló' }
                '''
            }
        }

        stage('4. Verificar monitoreo') {
            steps {
                powershell '''
                    $listo = $false
                    for ($i = 0; $i -lt 30; $i++) {
                        try {
                            $app = Invoke-WebRequest 'http://localhost:8000/health' -UseBasicParsing -TimeoutSec 5
                            $prom = Invoke-RestMethod 'http://localhost:9090/api/v1/query?query=up' -TimeoutSec 5
                            $kibana = Invoke-WebRequest 'http://localhost:5601/api/status' -UseBasicParsing -TimeoutSec 5
                            $jobs = @($prom.data.result | Where-Object { $_.value[1] -eq '1' } | ForEach-Object { $_.metric.job })
                            if ($app.StatusCode -eq 200 -and $kibana.StatusCode -eq 200 -and
                                $jobs -contains 'flask_service' -and $jobs -contains 'prometheus') {
                                $listo = $true
                                break
                            }
                        } catch { }
                        Start-Sleep -Seconds 10
                    }
                    if (-not $listo) { throw 'La app, Kibana o un target de Prometheus no está listo' }
                    Write-Host 'Entrega simulada correcta: app disponible y targets UP.'
                '''
            }
        }

        stage('5. Verificar acceso') {
            steps {
                powershell '''
                    try {
                        Invoke-WebRequest 'http://localhost:9200/' -UseBasicParsing -TimeoutSec 5 -ErrorAction Stop | Out-Null
                        throw 'Elasticsearch permite acceso sin contraseña'
                    } catch {
                        if ([int]$_.Exception.Response.StatusCode -ne 401) { throw }
                    }
                    Write-Host 'Control de seguridad correcto: Elasticsearch exige autenticación.'
                '''
            }
        }
    }
}
