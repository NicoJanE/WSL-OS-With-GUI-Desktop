## For  what
This file can keep track of your created and exported WSL distributions

| What | File |
|:--------|:------------| 
|Initial Debian with Mate GUI desktop | ***distro_export_debian-mate-GUI_initial.tar***

<br>

### Export 
`wsl --export <DistributionName> <PathToExportFile.tar>`

### Import
`wsl --import <NewDistributionName> <InstallLocation> <TarFilePath>`


🔄 Notes:
- You can export running or stopped distributions.
- The imported distribution is independent and can have a different name.
- The imported instance won’t have your previous default user unless you reconfigure it manually wit