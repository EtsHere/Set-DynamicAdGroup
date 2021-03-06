# Set-DynamicAdGroup
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

#Usage

$GroupList = @(
    @{ToGroup="new-users";FromGroup="";FromOU="";FromFilter="Name -like 'Adam *'"}
    @{ToGroup="all_admins";FromGroup="sql-admins","sccm-admins";FromOU="";FromFilter=""} | % { New-Object object | Add-Member -NotePropertyMembers $_ -PassThru | Select ToGroup,FromGroup,FromOU,FromFilter})

ForEach ($item in $GroupList)
    {
    Set-DynamicADGroup -ToGroup $item.ToGroup -FromGroup $item.FromGroup -FromOU $item.FromOU -FromFilter $item.FromFilter -WhatIf -Verbose
    $item = $null
    }
