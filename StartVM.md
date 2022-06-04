workflow StartVM
{
	(
		{
		Import-module 'az.accounts',
		Import-module 'az.compute'
		}
	)
	Connect-AzAccount -Identity
	Set-AzContext -SubscriptionId "Id da sua Subscription"
	$resultList = @()
    $retryList = @()
    $retryListTemp = @()
    try
    {
        $VMs = Get-AzVM -ResourceGroupName "Nome do seu ResourceGroup" | Where-Object { $_.Tags["nome da tag"] -eq "valor da tag" }
        
        do
        {
            $retryListTemp = @()
            ForEach -Parallel ($Vm in $VMs) 
            {
                $obj = $workflow:retryList | Where-Object { $_.ResourceGroup -eq $Vm.ResourceGroupName -and $_.Name -eq $Vm.Name }
                $Status = $null
                if ($obj -eq $null)
                {
                    $obj= @{
                                "ResourceGroup" = $Vm.ResourceGroupName;
                                "Name" = $Vm.Name;
                                "ActionResult" = "";
                                "RetryAttempts" = 0;
                                "Output" = ""
                            }
                }
                else
                {
                    $obj = InlineScript { $o = $using:obj; $o.RetryAttempts += 1; $o}
                }
                    if ($obj.RetryAttempts -gt 1)
                    {
                        $obj = InlineScript { $o = $using:obj; $o.RetryAttempts -= 1; $o}
                        Write-Output ("Failed starting VM: " + $Vm.Name)
                        $workflow:resultList += $obj
                    }
                    else
                    {
                        Write-Output ("Starting VM: " + $Vm.Name)
                        try
                        {
                            $Status = Start-AzVM -Name $Vm.Name -ResourceGroupName $Vm.ResourceGroupName -ErrorAction Stop
                            if ($Status.Error -eq $null)
                            {
                                $obj = InlineScript { $o = $using:obj; $o.ActionResult = "Started"; $o.Output = $using:Status.Error; $o}
                                Write-Output ("VM started: " + $Vm.Name)
                                $workflow:resultList += $obj
                            }
                            else
                            {
                                $obj = InlineScript { $o = $using:obj; $o.ActionResult = "Failed starting"; $o.Output = $using:Status.Error; $o}
                                $workflow:retryListTemp += $obj
                                Write-Output ("Error starting VM: " + $Vm.Name)
                                Write-Output ("Error message: " + $Status.Error)
                            }
                        }
                        catch
                        {
                            $obj = InlineScript { $o = $using:obj; $o.ActionResult = "Failed starting"; $o.Output = $using:_.ErrorRecord; $o}
                            $workflow:retryListTemp += $obj
                            Write-Output ("Error starting VM: " + $Vm.Name)
                            Write-Output ("Error message: " + $_.ErrorRecord)
                        }
                    }
                }
            $VMs = $VMs | Where-Object { $_.Name -notin ($resultList | select @{n="Name"; e={$_.Name} } ).Name }
            
            $retryList = $retryListTemp
        }
        while ($retryList.Count -gt 0)
    }
    catch
    {
        Write-Error -Message $_.Exception
        throw $_.Exception
    }
    if ($resultList.Count -eq 0)
    {
        Write-Output "No VMs scheduled to start at this time."
        return
    }
    else
    {
        Write-Output "Task completed, check logs"
    }
}