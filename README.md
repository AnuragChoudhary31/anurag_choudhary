# Configures local test source and local PowerShell repository.
parameters:
- name: sourceDir
  type: string
- name: localhostWebServerArgs
  type: string
- name: signingCertOutDir
  type: string
  default: $(Agent.TempDirectory)

steps:
  - pwsh: |
      $newCertArguments = @{
        Type = "Custom"
        Subject = "CN=Microsoft Corporation, O=Microsoft Corporation, L=Redmond, S=Washington, C=US"
        KeyUsage = "DigitalSignature"
        TextExtension = @("2.5.29.37={text}1.3.6.1.5.5.7.3.3", "2.5.29.19={text}")
        CertStoreLocation = "Cert:\CurrentUser\My"
      }
      $cert = New-SelfSignedCertificate @newCertArguments

      $certPfxPath = Join-Path $(Agent.TempDirectory) TestSigningCert.pfx
      $certCerPath = Join-Path ${{ parameters.signingCertOutDir }} TestSigningCert.cer
      $certPassword = (New-Guid).ToString()
      $certSecurePassword = ConvertTo-SecureString $certPassword -AsPlainText

      Export-PfxCertificate -Cert $cert -FilePath $certPfxPath -Password $certSecurePassword
      Export-Certificate -Cert $cert -FilePath $certCerPath

      Write-Host "##vso[task.setvariable variable=TestSigningCert.PfxPath;]$certPfxPath"
      Write-Host "##vso[task.setvariable variable=TestSigningCert.CerPath;]$certCerPath"
      Write-Host "##vso[task.setvariable variable=TestSigningCert.Password;]$certPassword"
    displayName: Create test codesigning cert
    condition: succeededOrFailed()

  - pwsh: |
      $httpsCertPath = Join-Path $(Agent.TempDirectory) HttpsCert.pfx
      $httpsCertPassword = (New-Guid).ToString()
      dotnet dev-certs https --export-path $httpsCertPath --password $httpsCertPassword

      $securePassword = ConvertTo-SecureString $httpsCertPassword -AsPlainText
      Import-PfxCertificate -FilePath $httpsCertPath -Password $securePassword -CertStoreLocation Cert:\LocalMachine\Root

      Write-Host "##vso[task.setvariable variable=HttpsCert.Path;]$httpsCertPath"
      Write-Host "##vso[task.setvariable variable=HttpsCert.Password;]$httpsCertPassword"
    displayName: Create and install localhost HTTPS cert
    condition: succeededOrFailed()

  - pwsh: ${{ parameters.sourceDir }}\src\LocalhostWebServer\Run-LocalhostWebServer.ps1 -CertPath $(HttpsCert.Path) -CertPassword $(HttpsCert.Password) -OutCertFile $(Agent.TempDirectory)\servercert.cer ${{ parameters.localhostWebServerArgs }}
    displayName: Launch LocalhostWebServer
    condition: succeededOrFailed()

  - task: PowerShell@2
    displayName: Setup Local PS Repository
    condition: succeededOrFailed()
    inputs:
      filePath: '${{ parameters.sourceDir }}\src\AppInstallerCLIE2ETests\TestData\Configuration\Init-TestRepository.ps1'
      arguments: '-Force'
      pwsh: true
