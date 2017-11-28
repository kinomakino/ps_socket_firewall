# ps_socket_firewall
Script Powershell para levantar un socket y detectar una conexión. Al detectar banea en firewall y manda correo.



[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Drawing") 
[void] [System.Reflection.Assembly]::LoadWithPartialName("System.Windows.Forms") 

$objForm = New-Object System.Windows.Forms.Form 
$objForm.Text = "Baneador IP.kinomakino"
$objForm.Size = New-Object System.Drawing.Size(300,200) 
$objForm.StartPosition = "CenterScreen"

$objForm.KeyPreview = $True
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Banear") 
    {$x=$objTextBox.Text;$objForm.Close()}})
$objForm.Add_KeyDown({if ($_.KeyCode -eq "Cancelar") 
    {$objForm.Close()}})

$OKButton = New-Object System.Windows.Forms.Button
$OKButton.Location = New-Object System.Drawing.Size(75,120)
$OKButton.Size = New-Object System.Drawing.Size(75,23)
$OKButton.Text = "OK"
$OKButton.Add_Click({$x=$objTextBox.Text;$objForm.Close()})
$objForm.Controls.Add($OKButton)

$CancelButton = New-Object System.Windows.Forms.Button
$CancelButton.Location = New-Object System.Drawing.Size(150,120)
$CancelButton.Size = New-Object System.Drawing.Size(75,23)
$CancelButton.Text = "Cancela"
$CancelButton.Add_Click({$objForm.Close()})
$objForm.Controls.Add($CancelButton)

$objLabel = New-Object System.Windows.Forms.Label
$objLabel.Location = New-Object System.Drawing.Size(10,20) 
$objLabel.Size = New-Object System.Drawing.Size(280,20) 
$objLabel.Text = "Introduce el puerto que quieres monitorizar:"
$objForm.Controls.Add($objLabel) 

$objTextBox = New-Object System.Windows.Forms.TextBox 
$objTextBox.Location = New-Object System.Drawing.Size(10,40) 
$objTextBox.Size = New-Object System.Drawing.Size(260,20) 
$objForm.Controls.Add($objTextBox) 

$objForm.Topmost = $True

$objForm.Add_Shown({$objForm.Activate()})
[void] $objForm.ShowDialog()

$x = $objTextBox.Text
#Toda la parte de arriba es para mostrar el textbox y almacenarlo en $x

$Honeypot = [System.Net.Sockets.TcpListener][int]$x;
$Honeypot.Start();
#Abre el socket con el puerto indicado en el Textbox
while($true)
{
sleep -s 300
#comprueba cada 5minutos si ha habido intento de conexión mediante el log del Firewall.

$logPath="C:\Windows\System32\LogFiles\Firewall\pfirewall.log"
$blackListPath= "C:\Windows\System32\LogFiles\Firewall\blacklist.txt"
$blacklist = Get-Content $blackListPath

$header=(Select-String -Path $logPath -Pattern "^#Fields").Line.Split(" ") | select -Skip 1
$ips=Get-Content $logPath | select -Skip 4 | 
 ConvertFrom-Csv -Delimiter " " -Header $header |
 where {$_."dst-port" -eq $x -and $_.path -eq "RECEIVE" -and $blacklist -notcontains $_."src-ip"} |
 select -ExpandProperty "src-ip" 
$ips
$ips | Add-Content $blacklistPath
foreach ($ip in $ips){
  New-NetFirewallRule -DisplayName "Ip Bloqueada $ip" -Direction Inbound -Action Block -RemoteAddress $ip
   #si encuentra algún registro contra el puerto indicado, envia un correo.
                 $smtpserver = “smtp.gmail.com”
                 $msg = new-object Net.Mail.MailMessage
                 $smtp = new-object Net.Mail.SmtpClient($smtpServer )
                 $smtp.EnableSsl = $True
                 $smtp.Credentials = New-Object System.Net.NetworkCredential(“nick gmail sin @gmail, “clavegmail”); 
                 $msg.From = “correo origen”
                 $msg.To.Add(”correo destino”)
                 $msg.Subject = “Ip baneada”
                 $msg.Body = “La ip $ip ha sido baneada por intento de conexión al honeypot”
                 $smtp.Send($msg)
                      }
                  
                        }
