[![Build status](https://ci.appveyor.com/api/projects/status/tsdbv0hgrxvmbo5y/branch/master?svg=true)](https://ci.appveyor.com/project/PowerShell/xhyper-v/branch/master)

# xHyper-V

The **xHyper-V** DSC module configures and manages a Hyper-V host using the **xVhd**, **xVMHyperV**, **xVMSwitch**, **xVhdFileDirectory** resources. 

## Contributing
Please check out common DSC Resources [contributing guidelines](https://github.com/PowerShell/DscResource.Kit/blob/master/CONTRIBUTING.md).


## Resources

* **xVhd** manages VHDs in a Hyper-V host. 
* **xVMHyperV** manages VMs in a Hyper-V host.
* **xVMSwitch** manages virtual switches in a Hyper-V host. 
* **xVhdFileDirectory** manages files or directories in a VHD. 
You can use it to copy files/folders to the VHD, remove files/folders from a VHD, and change attributes of a file in a VHD (e.g. change a file attribute to 'ReadOnly' or 'Hidden').
This resource is particularly useful when bootstrapping DSC Configurations into a VM. 

### xVhd

* **Name**: The desired VHD file name 
* **Path**: The desired Path where the VHD will be created
* **ParentPath**: Parent VHD file path, for differencing disk 
* **MaximumSizeBytes**: Maximum size of VHD to be created 
* **Generation**: Virtual disk format: { Vhd | VHDx }
* **Ensure**: Ensures that the VHD is Present or Absent 

### xVMHyperV

* **Name**: The desired VM name 
* **VhdPath**: The desired VHD associated with the VM 
* **SwitchName**: Virtual switch associated with the VM 
* **State**: State of the VM: { Running | Paused | Off }
* **Path**: Folder where the VM data will be stored
* **Generation**: Associated Virtual disk format { Vhd | Vhdx }
* **StartupMemory**: Startup RAM for the VM 
* **MinimumMemory**: Minimum RAM for the VM. 
Setting this property enables dynamic memory.
* **MaximumMemory**: Maximum RAM for the VM.
Setting this property enables dynamic memory.
* **MACAddress**: MAC address of the VM
* **ProcessorCount**: Processor count for the VM
* **WaitForIP**: If specified, waits for the VM to get valid IP address
* **RestartIfNeeded**: If specified, will shutdown and restart the VM as needed for property changes
* **Ensure**: Ensures that the VM is Present or Absent

### xVMSwitch

* **Name**: The desired VM Switch name 
* **Type**: The desired type of switch: { External | Internal | Private }
* **NetAdapterName**: Network adapter name for external switch type
* **AllowManagementOS**: Specify if the VM host has access to the physical NIC
* **Ensure**: Ensures that the VM Switch is Present or Absent 

### xVhdFileDirectory

* **Vhdpath**: Path to the VHD 
* **FileDirectory**: The FileDirectory objects to copy to the VHD (as used in the "File" resource). 
Please see the Examples section for more details. 

## Versions

### 2.1

* Added logic to automatically adjust VM’s startup memory when only minimum and maximum memory is specified in configuration
* Fixed the issue that a manually stopped VM cannot be brought back to running state with DSC

### 2.0
* Added xVhdFileDirectory Resource
* Allowed name to be specified with the extension in xVhd (e.g. the Vhd name could either be "sample" or "sample.vhd")
* When a VHD cannot be removed because it is already being used by another process, an error will be thrown.

### 1.0.0.0

* Initial release with the following resources
    - xVhd 
    - xVMHyperV 
    - xVMSwitch  


## Examples

### End-to-End Example

In this example, we ensure that an internal VM Switch and a VM made from VHD with an unattended.xml file are present.

```powershell
# End to end sample for x-Hyper-v
Configuration Sample_EndToEndXHyperV_RunningVM
{

    param
    (
        [Parameter(Mandatory)]
        $baseVhdPath,

        [Parameter(Mandatory)]
        $name,
        [Parameter(Mandatory)]
        [validatescript({Test-Path $_})]
        $unattendedFilePathToCopy

    )

    Import-DscResource -module xHyper-V

    # Create a switch to be used by the VM
    xVMSwitch switch
    {
        Name =  "Test-Switch"
        Ensure = "Present"      
        Type = "Internal"
    }

    # Create new VHD file.
    xVHD NewVHD1
    {

            Ensure         = "Present"
            Name           = $name
            Path           = (Split-Path $baseVhdPath)
            Generation     = "vhd"
            ParentPath     =  $baseVhdPath

    }

    # Customize VHD by copying a folders/files to the VHD before a VM can be created
    # Example below shows copying unattended.xml before a VM can be created
    xVhdFile CopyUnattendxml
    {
            VhdPath =  $vhdPath
            FileDirectory =  MSFT_xFileDirectory {
                               SourcePath = $unattendedFilePathToCopy
                               DestinationPath = "unattended.xml"
                               }

    }

    # create the testVM out of the vhd.
    xVMHyperV testvm
    {
        Name = "$($name)_vm"
        SwitchName = "Test-Switch"
        VhdPath = Join-path (Split-Path $baseVhdPath) "$name.vhd"
        ProcessorCount = 2
        MaximumMemory  = 1GB
        MinimumMemory = 512MB
        RestartIfNeeded =  "TRUE"
        DependsOn = "[xVHD]NewVHD1","[xVMSwitch]switch","[xVhdFile]CopyUnattendxml"
        State = "Running"

    }

}
```

### Create a new VHD

This configuration will create a new VHD on Hyper-V host.

```powershell
Configuration Sample_xVHD_NewVHD
{
    param
    (
        [Parameter(Mandatory)]
        [string]$Name,

        [Parameter(Mandatory)]
        [string]$Path,

        [Parameter(Mandatory)]
        [Uint64]$MaximumSizeBytes,

        [ValidateSet("Vhd","Vhdx")]
        [string]$Generation = "Vhd",

        [ValidateSet("Present","Absent")]
        [string]$Ensure = "Present"        
    )

    Import-DscResource -module xHyper-V

    Node localhost
    {
        xVHD NewVHD
        {
            Ensure           = $Ensure
            Name             = $Name
            Path             = $Path
            Generation       = $Generation
            MaximumSizeBytes = $MaximumSizeBytes
        }
    }
}
```

### Create a differencing VHD

This configuration will create a differencing VHD, given a parent VHD, on a Hyper-V host.

```powershell
Configuration Sample_xVhd_DiffVHD
{
    param
    (
        [Parameter(Mandatory)]
        [string]$Name,

        [Parameter(Mandatory)]
        [string]$Path,

        [Parameter(Mandatory)]
        [string]$ParentPath,

        [ValidateSet("Vhd","Vhdx")]
        [string]$Generation = "Vhd",

        [ValidateSet("Present","Absent")]
        [string]$Ensure = "Present"    
    )

    Import-DscResource -module xHyper-V

    Node localhost
    {
        xVHD DiffVHD
        {
            Ensure     = $Ensure
            Name       = $Name
            Path       = $Path
            ParentPath = $ParentPath
            Generation = $Generation
        }
    }
}
```

### Create a VM for a given VHD

This configuration will create a VM, given a VHD, on Hyper-V host.

```powershell
Configuration Sample_xVMHyperV_Simple
{
    param
    (
        [string[]]$NodeName = 'localhost',

        [Parameter(Mandatory)]
        [string]$VMName,

        [Parameter(Mandatory)]
        [string]$VhdPath        
    )

    Import-DscResource -module xHyper-V

    Node $NodeName
    {
        # Install HyperV feature, if not installed - Server SKU only
        WindowsFeature HyperV
        {
            Ensure = 'Present'
            Name   = 'Hyper-V'
        }

        # Ensures a VM with default settings
        xVMHyperV NewVM
        {
            Ensure     = 'Present'
            Name       = $VMName
            VhdPath    = $VhdPath
            Generation = $VhdPath.Split('.')[-1]
            DependsOn  = '[WindowsFeature]HyperV'
        }
    }
} 
```

### Create a VM with dynamic memory for a given VHD

This configuration will create a VM with dynamic memory settings, given a VHD, on Hyper-V host.

```powershell
Configuration Sample_xVMHyperV_DynamicMemory
{
    param
    (
        [string[]]$NodeName = 'localhost',

        [Parameter(Mandatory)]
        [string]$VMName,

        [Parameter(Mandatory)]
        [string]$VhdPath,

        [Parameter(Mandatory)]
        [Uint64]$StartupMemory,

        [Parameter(Mandatory)]
        [Uint64]$MinimumMemory,

        [Parameter(Mandatory)]
        [Uint64]$MaximumMemory
    )

    Import-DscResource -module xHyper-V

    Node $NodeName
    {
        # Install HyperV feature, if not installed - Server SKU only
        WindowsFeature HyperV
        {
            Ensure = 'Present'
            Name   = 'Hyper-V'
        }

        # Ensures a VM with dynamic memory
        xVMHyperV NewVM
        {
            Ensure        = 'Present'
            Name          = $VMName
            VhdPath       = $VhdPath
            Generation    = $VhdPath.Split('.')[-1]
            StartupMemory = $StartupMemory
            MinimumMemory = $MinimumMemory
            MaximumMemory = $MaximumMemory
            DependsOn     = '[WindowsFeature]HyperV'
        }
    }
}
```

### Create a VM with dynamic memory, network interface and processor count for a given VHD

This configuration will create a VM with dynamic memory, network interface and processor count settings, given a VHD, on Hyper-V host.

```powershell
Configuration Sample_xVMHyperV_Complete
{
    param
    (
        [string[]]$NodeName = 'localhost',

        [Parameter(Mandatory)]
        [string]$VMName,

        [Parameter(Mandatory)]
        [string]$VhdPath,

        [Parameter(Mandatory)]
        [Uint64]$StartupMemory,

        [Parameter(Mandatory)]
        [Uint64]$MinimumMemory,

        [Parameter(Mandatory)]
        [Uint64]$MaximumMemory,

        [Parameter(Mandatory)]
        [String]$SwitchName,

        [Parameter(Mandatory)]
        [String]$Path,

        [Parameter(Mandatory)]
        [Uint32]$ProcessorCount,

        [ValidateSet('Off','Paused','Running')]
        [String]$State = 'Off',

        [Switch]$WaitForIP
    )

    Import-DscResource -module xHyper-V

    Node $NodeName
    {
        # Install HyperV feature, if not installed - Server SKU only
        WindowsFeature HyperV
        {
            Ensure = 'Present'
            Name   = 'Hyper-V'
        }

        # Ensures a VM with all the properties
        xVMHyperV NewVM
        {
            Ensure          = 'Present'
            Name            = $VMName
            VhdPath         = $VhdPath
            SwitchName      = $SwitchName
            State           = $State
            Path            = $Path
            Generation      = $VhdPath.Split('.')[-1]
            StartupMemory   = $StartupMemory
            MinimumMemory   = $MinimumMemory
            MaximumMemory   = $MaximumMemory
            ProcessorCount  = $ProcessorCount
            MACAddress      = $MACAddress
            RestartIfNeeded = $true
            WaitForIP       = $WaitForIP 
            DependsOn       = '[WindowsFeature]HyperV'
        }
    }
}
```

### Create an internal VM Switch

This configuration will create an internal VM Switch, on Hyper-V host.

```powershell
Configuration Sample_xVMSwitch_Internal
{
    param
    (
        [string[]]$NodeName = 'localhost',

        [Parameter(Mandatory)]
        [string]$SwitchName
    )

    Import-DscResource -module xHyper-V

    Node $NodeName
    {
        # Install HyperV feature, if not installed - Server SKU only
        WindowsFeature HyperV
        {
            Ensure = 'Present'
            Name   = 'Hyper-V'
        }

        # Ensures a VM with default settings
        xVMSwitch InternalSwitch
        {
            Ensure         = 'Present'
            Name           = $SwitchName
            Type           = 'Internal'
            DependsOn      = '[WindowsFeature]HyperV'
        }
    }
}
```

### Create an external VM Switch

This configuration will create an external VM Switch, on Hyper-V host.

```powershell
Configuration Sample_xVMSwitch_External
{
    param
    (
        [string[]]$NodeName = 'localhost',

        [Parameter(Mandatory)]
        [string]$SwitchName,

        [Parameter(Mandatory)]
        [string]$NetAdapterName        
    )

    Import-DscResource -module xHyper-V

    Node $NodeName
    {
        # Install HyperV feature, if not installed - Server SKU only
        WindowsFeature HyperV
        {
            Ensure = 'Present'
            Name   = 'Hyper-V'
        }

        # Ensures a VM with default settings
        xVMSwitch ExternalSwitch
        {
            Ensure         = 'Present'
            Name           = $SwitchName
            Type           = 'External'
            NetAdapterName = $NetAdapterName 
            DependsOn      = '[WindowsFeature]HyperV'
        }
    }
}
```

### Copy a file or folder into a VHD  

```powershell
Configuration xVhdD_CopyFileOrFolder
{

     Param(
     [Parameter(Mandatory=$true, Position=0)]
     [validatescript({Test-Path $_})]
     $vhdPath,
     [Parameter(Mandatory=$true)]
     [validatescript({Test-Path $_})]
     $itemToCopy,
     [Parameter(Mandatory=$true)]
     $relativeDestinationPath
    )

        Import-DscResource -moduleName xHyper-V

        xVhdFile FileCopy
        {
            VhdPath =  $vhdPath
            FileDirectory =  MSFT_xFileDirectory {
                               SourcePath = $itemToCopy
                               DestinationPath = $relativeDestinationPath
                               }

        }

}
```
### Change an Attribute for a File  

```powershell 
Configuration ChangeAttribute
{
    param(
      [Parameter(Mandatory=$true, Position=0)]
      [validatescript({Test-Path $_})]
      $vhdPath,
      [Parameter(Mandatory=$true)]
      $relativeDestinationPath,
      [ValidateSet ("Archive", "Hidden",  "ReadOnly", "System" )] $attribute
    )

    Import-DscResource -moduleName xHyper-V
      xVhdFile Change-Attribute
        {
            VhdPath =  $vhdPath
            FileDirectory =  MSFT_xFileDirectory {
                               DestinationPath = $relativeDestinationPath
                               Attributes  = $attribute
                               }

        }
}
```
