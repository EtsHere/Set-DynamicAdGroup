# Set-DynamicAdGroup
Keep users in a group dynamically.


#Requires -Modules ActiveDirectory
#Requires -Version 5.0

Function Set-DynamicAdGroup
    {  
    <#
    .SYNOPSIS
    Keep users in a group dynamically.

    .DESCRIPTION
    Set-DynamicAdGroup searches users from OU, groups or ad Filter and adds found users to the relevant To Group..
    Group additions and removals are dynamic. The change accurs only on changes to the original group.

    .EXAMPLE
    Set-DynamicADGroup -ToGroup "g_admins" -FromGroup "domain_admins","main_administrators" -WhatIf -Verbose

    .EXAMPLE
    Set-DynamicADGroup -ToGroup "g_admins" -FromGroup "domain_admins","main_administrators" -FromOU "OU=Admins,OU=MY,DC=comp,DC=local" -WhatIf -Verbose

    .PARAMETER ToGroup
    Groupp that needs to be up to date dynamically.

    .PARAMETER FromGroup
    Groups SamAccountName

    .PARAMETER FromOU
    Organization Unit DistinguishedName

    .PARAMETER FromFilter
    Get-ADUser suported -Filter

    .LINK
    https://
    #>

[CmdletBinding(SupportsShouldProcess)]
Param(
    [Parameter(Mandatory)]
    [ValidateScript({Get-ADGroup $_})]
    [string]$ToGroup,
    [string[]]$FromGroup,
    [string[]]$FromOU,
    [string[]]$FromFilter
    ) #end param

    Try
        {
        Write-Host "Add users to group:" $ToGroup -ForegroundColor Red
        Write-Host "Search users from:" $FromGroup $FromOU $FromFilter -Separator "`n" -ForegroundColor Yellow

        #--- Users who should be in a specific group. ---
        $AnyUsers = @()
        $AnyUsers = Get-ADGroupMember -Identity $ToGroup -Recursive
        $Users = @()

        #--- Search Users From OU, Groups and using Filter.. ---
        If(($FromGroup.Count -eq 1) -and ($FromGroup -ne "")) #When one group
            {
            $Users += Get-ADGroupMember "$FromGroup" -Recursive
            }
        If($FromGroup.Count -gt 1) #When more than one group
            {
            ForEach($item in $FromGroup)
                {
                $Users += Get-ADGroupMember $item -Recursive
                } #end of ForEach
            }
        If(($FromOU -like "*=*") -and ($FromOU.Count -eq 1)) #When one OU
            {
            $Users += Get-ADUser -SearchBase $FromOU -Filter {Enabled -eq $true}
            }
        If(($FromOU -like "*=*") -and ($FromOU.Count -gt 1)) #When more than one OU
            {
            ForEach($item in $FromOU)
                {
                $Users += Get-ADUser -SearchBase $item -Filter {Enabled -eq $true}
                } #end of ForEach
            }
        If(($FromFilter.Count -eq 1) -and ($FromFilter -ne "")) #When one filter
            {
            [string]$Filter = '(' + $FromFilter + ') -and (Enabled -eq $true)'
            $Users += Get-ADUser -Filter $Filter
            $Filter = $null
            }
        If(($FromFilter.Count -gt 1)) #When More than one Filter
            {
            ForEach($FilterItem in $FromFilter)
                {
                [string]$Filter = '(' + $FilterItem + ') -and (Enabled -eq $true)'
                $Users += Get-ADUser -Filter $Filter
                $Filter = $null
                $FilterItem = $null
                } #end of ForEach
            }
            $Users = $Users | Select -Unique

            #--- Compare and add users from group ---
            $AllowedUsers = $Users | Where {$_.SamAccountName -notin $AnyUsers.SamAccountName}
            Write-Output "Lubatud: $($AllowedUsers.Count)"
            ForEach ($AllowedUser in $AllowedUsers)
                {
                Write-Verbose "Adding user: $($AllowedUser.SamAccountName)"
                Add-ADGroupMember -Identity $ToGroup -Members $AllowedUser #-WhatIf -Verbose
                }

            #--- Compare and remove users from group ---
            $RemovedUsers = $AnyUsers | Where {$_.SamAccountName -notin $Users.SamAccountName}
            Write-Output "Keelatud: $($RemovedUsers.Count)"
            ForEach ($RemovedUser in $RemovedUsers)
                {
                Write-Verbose "Removing user: $($RemovedUser.SamAccountName)"
                Remove-ADGroupMember $ToGroup -Members $RemovedUser -Confirm:$false #-WhatIf -Verbose
                }
        } #end of Try
    Catch
        {
        Write-Host "Error in script!" -ForegroundColor Cyan
        Write-Host $_.Exception.Message -ForegroundColor Cyan
        Write-Host $_.Exception.ItemName -ForegroundColor Cyan
        Write-Host $_.Exception -ForegroundColor Cyan
        } #end of Catch
    Finally
        {
        $Users = $null
        $AnyUsers = $null
        $RemovedUsers = $null
        $AllowedUsers = $null
        } #end of Finally
}#end of function Set-DynamicAdGroup



#Example use:
$GroupList = @(
    @{ToGroup="new-users";FromGroup="";FromOU="";FromFilter="Name -like 'Adam *'"}
    @{ToGroup="all_admins";FromGroup="sql-admins","sccm-admins";FromOU="";FromFilter=""} | % { New-Object object | Add-Member -NotePropertyMembers $_ -PassThru | Select ToGroup,FromGroup,FromOU,FromFilter})

ForEach ($item in $GroupList)
    {
    Set-DynamicADGroup -ToGroup $item.ToGroup -FromGroup $item.FromGroup -FromOU $item.FromOU -FromFilter $item.FromFilter -WhatIf -Verbose
    $item = $null
    }
