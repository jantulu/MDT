
$DeploymentFolder = "D:\Deployment"
$oscdimgPath = "c:\windows\system32\oscdimg.exe"
$OSArchitecture = "x64"
$date = get-date -UFormat "%m-%y"
$OSSlipstreamList = @(
 @{
  OperatingSystem = 'Windows Server 2019 Datacenter'
  Build = '1809'
  Index = 4
  IncludeDotNet = $False
 },
 @{
    OperatingSystem = 'Windows Server 2016 Datacenter'
    Build = '1607'
    Index = 4
    IncludeDotNet = $False
 }
 @{
    OperatingSystem = 'Windows 10 Professional'
    Build = '1903'
    Index = 6
    IncludeDotNet = $True
 } 
)
Foreach ($slipstream in $OSSlipStreamList) {
    $BaseFolder = "D:\slipstream\$($slipstream.OperatingSystem)"
    $UpdatesPath = "$($BaseFolder)\updates\*"
    $MountPath = "$($BaseFolder)\mount"
    $WimFile = "$($BaseFolder)\original\sources\install.wim"
    $servicingPath = "$($BaseFolder)\servicing\*"
    $dotNetPath = "$($BaseFolder)\dotNet\*"

# -- OS Specific
$OSName = $slipstream.OperatingSystem
$OSVersion = $slipstream.Build
$IndexNum = $slipstream.index
# -- End OS Specific

Write-Host "Running updates on $($slipstream.OperatingSystem)" -BackgroundColor Red -ForegroundColor Black



# Purge the folder contents before downloading, don't want multiple CU/Serv stacks
remove-item -path "$($BaseFolder)\servicing" -Recurse
remove-item -path "$($BaseFolder)\updates" -Recurse
remove-item -path "$($BaseFolder)\dotnet" -Recurse

New-Item -Path "$($BaseFolder)" -Name servicing -ItemType Directory | Out-Null
New-Item -Path "$($BaseFolder)" -Name updates -ItemType Directory | Out-Null
New-Item -Path "$($BaseFolder)" -Name dotNet -ItemType Directory | Out-Null

# Latest CU
set-location -path $BaseFolder\Updates
Get-LatestCumulativeUpdate -version $OSVersion| Where {$_.architecture -eq $OSArchitecture} |  save-latestupdate
# Latest Servicing Stack
set-location -path $BaseFolder\Servicing
Get-LatestServicingStackUpdate -version  $OSVersion | Where {$_.architecture -eq $OSArchitecture} | save-latestupdate

# loop through servicing/CU folders to apply their patches to the build.
Set-ItemProperty $WimFile -Name IsReadOnly -Value $false
DISM /Mount-Wim /WimFile:$WimFile /index:$IndexNum /Mountdir:$MountPath
$UpdateArray = Get-Item $UpdatesPath
$dotNetArray = Get-Item $dotNetPath
$servicingArray = Get-Item $servicingPath

ForEach ($Servicing in $ServicingArray) {
	DISM /image:$MountPath /Add-Package /Packagepath:$Servicing
	Start-Sleep -s 10
}

ForEach ($Updates in $UpdateArray) {
	DISM /image:$MountPath /Add-Package /Packagepath:$Updates
	Start-Sleep -s 10
}

if ($slipstream.IncludeDotNet -eq $True) {
    write-host "Including .NET CU's"
# Latest .Net Update
    set-location -path $BaseFolder\dotNet
    Get-LatestNetFrameworkUpdate | Where-Object {$_.Architecture -eq $OSArchitecture} | Where-Object {$_.Version -eq $OSVersion} | Save-LatestUpdate    
    ForEach ($dotNet in $dotNetArray) {
        DISM /image:$MountPath /Add-Package /Packagepath:$dotNet
        Start-Sleep -s 10
    }
}
DISM /Unmount-image /Mountdir:$MountPath /commit
DISM /Cleanup-Wim

set-location -path $BaseFolder\original
## REM Below compiles the folder into a bootable ISO if needed.
& $($oscdimgPath) -bootdata:"2#p0,e,bboot\Etfsboot.com#pEF,e,befi\Microsoft\boot\Efisys.bin" -u1 -udfver102 "$BaseFolder\original" "$BaseFolder\$($OSName)_$($OSVersion)_$($date).iso"
#Import Windows 10 pro into MDT

Add-PSSnapIn Microsoft.BDD.PSSnapIn
 $date = get-date -UFormat "%m-%y"
# Create $Basefolder\wim to store the install.wim.
New-Item -Path "$($BaseFolder)" -Name WIM -ItemType Directory | Out-Null
 
Write-Host "Importing $($BaseFolder)\$($OSName)_$($OSVersion)_$($date).iso." -BackgroundColor Green -ForegroundColor Black
$date = get-date -UFormat "%y-%m-%d"
$ISO = Get-ChildItem -path $BaseFolder -Recurse -Include *.iso 
    foreach ($ISOName in $ISO) {
        $ISOImagePath = Mount-DiskImage -PassThru "$($BaseFolder)\$($ISOName.Name)"
    }
$GetLetter = Get-Volume -DiskImage $ISOImagePath
 
Write-Host "Copying $($GetLetter.DriveLetter + ":\sources\install.wim") to $($BaseFolder + "\WIM")..." -BackgroundColor Green -ForegroundColor Black
Copy-Item -Path ($GetLetter.DriveLetter + ":\sources\install.wim") -Destination ($BaseFolder + "\WIM")
 
# Set location to same as WIM file folder, create emtpy/temp folder.
$Loc = "$($BaseFolder)\WIM"
Set-Location -Path $Loc
New-Item -Name Empty -ItemType Directory | Out-Null
New-Item -Name Temp -ItemType Directory | Out-Null
 
Write-Host "Using $($OSName)" -BackgroundColor Green -ForegroundColor Black
# Create capture WIM in \temp.
Write-Host "Creating new WIM in $Loc\Temp\install.wim..."  -BackgroundColor Green -ForegroundColor Black
New-WindowsImage -ImagePath "$Loc\Temp\$($OSName).wim" -CapturePath "$Loc\Empty" -Name $OSName -CompressionType max | Out-Null
$TempWIM = "$Loc\Temp\$($OSname).wim"
 
# Export install.wim to empty WIM.
 
Write-Host "Exporting $($Loc)\install.wim with $($OSName) $($OSVersion) to $($Loc)\Temp\$($OSName).wim..."  -BackgroundColor Green -ForegroundColor Black
Export-WindowsImage -SourceImagePath "$Loc\install.wim" -SourceIndex $IndexNum -DestinationImagePath $($TempWIM) -DestinationName $OSName -CompressionType max | Out-Null
 
# Clear ImageIndex where ImageSize -eq 0.
 
$TempImage = Get-WindowsImage -ImagePath $TempWIM
$ImageSize = $TempImage | Where-Object { $_.ImageSize -eq '0'}
Remove-WindowsImage -ImagePath $TempWIM -Index $ImageSize.ImageIndex -InformationAction SilentlyContinue | Out-Null
 
# Import the WIM into MDT.
$WinInfo = Get-WindowsImage -ImagePath $TempWIM -Index 1
$WinVer = "$($OSName) $($OSVersion) $($OSArchitecture)"
 
switch -Wildcard ($WinVer)
    {
    "Windows 10*" { $WinVer = "Windows 10" }
    "Windows 8*"  { $WinVer = "Windows 8" }
    "Windows 7*"  { $WinVer = "Windows 7" }
    "Windows Server 2016*"  { $WinVer = "Windows Server 2016" }
    "Windows Server 2019*"  { $WinVer = "Windows Server 2016" }
    default { Write-Host "No valid Windows version." -ForegroundColor Red } 
    }
# Set PSDrive, create new if unavailable.
$PSDrive = "DS001:"
if (!(Test-Path -Path $PSDrive)){
 Write-Host "Creating new PSDrive..."  -BackgroundColor Green -ForegroundColor Black
 New-PSDrive -Name "DS001" -PSProvider MDTProvider -Root "$($DeploymentFolder)" | Out-Null
 }else{
 Write-Host "PSDrive is available."
 }
# Import exported WIM's in to MDT.
 
Write-Host "Importing $($TempWIM) into MDT..." -BackgroundColor Green -ForegroundColor Black
Import-MDTOperatingSystem -Path "$($PSDrive)\Operating Systems" -SourceFile $($TempWIM) -DestinationFolder "$($OSName) - $($WinInfo.Version.ToString())" | Out-Null
 
# Dismount mounted ISO.
Write-Host "Dismounting the ISO $($ISO.Name)" -BackgroundColor Green -ForegroundColor Black
Dismount-DiskImage  "$($BaseFolder)\$($ISOName.Name)"


#remove-item "$($BaseFolder)\$($ISOName.Name)"

# Remove C:\WIM folder after import.
Set-Location -Path $BaseFolder
Write-Host "Removing the folder and contents: $Loc ..." -BackgroundColor Green -ForegroundColor Black
Remove-Item $Loc -Recurse -Force

# End Import
# Clean up the downloaded data.
remove-item -path "$($BaseFolder)\servicing" -Recurse
remove-item -path "$($BaseFolder)\updates" -Recurse
remove-item -path "$($BaseFolder)\dotnet" -Recurse

}
