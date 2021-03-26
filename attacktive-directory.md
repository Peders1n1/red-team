---
layout: default
---
# Attacktive Directory - 2021

- ### Kerberos abuse
- ### Brute force in Hash
- ### Pass The Hash

## Recon

Começo realizando um scan com o nmap para validar as portas abertas e os serviços em execução
```sh
sudo nmap -sV -sC -T4 -Pn attacktive.thm -v -oN nmap.scan
```
Percebemos que se trada de um Windows Server com o mome do dominio ***spookysec.local***

![Scan nmap](./images/walkthroughs/attacktive-directory/img.attacktive-2.png)

Agora começa a ficar mais intereessante, pois como se trata de um DC inicialmente podemos procurar por diversos vetores de ataque

Como notamos na fase de Recon, o servidor possui um serviço chamado Kerberos

Kerberos fornece um serviço de autenticação através de "tickets" tanto de usuários, como serviços e aplicações, sendo assim, podemos tentar fazer uma enumeração dos usuarios validos no dominio atraves de brute force

Para isso, iremos utilizar uma ferramenta chamada kerbrute https://github.com/ropnop/kerbrute ela consegue se comunicar com o protocolo do kerberos e realizar as consultas de usuario.

Mas antes, vamos escovar uns bits e ver como funciona!

Primeiro vamos enviar um usuario que provavelmente não existe no dominio como por exemplo "imnotexist" e ver como o protocolo se comporta, para isso vou utilizar o Wireshark

![Wireshark-1](./images/walkthroughs/attacktive-directory/img.attacktive-3.png)

Após realizar a query, notamos que o protocolo responsavel pelo Kerberos é o KRB5 na porta 88 UDP, então ele envia uma flag "AS-REQ" e o servidor responde com "KRB:Error KRB5KDC_ERR_C_PRINCIPAL_UNKNOM" porque o usuario não existe no dominio.

Vamos fazer uma query agora com um usuario valido para verificarmos se o protocolo se comporta de forma diferente.
Para isso, vou utilizar um usuario padrão que na maioria das vezes fica ativo no dominio, o "administrator".

![Wireshark-2](./images/walkthroughs/attacktive-directory/img.attacktive-4.png)

E boom! O protocolo se comporta de forma diferente, respondendo com a flag "KRB:Error KRB5KDC_ERR_PREAUTH_REQUIRED" informando que o usuario é valido mas necessita se autenticar.

Sendo assim, após realizar a PoC notamos que sim, é possivel realizar ataques de enumeração de usuarios apartir do protocolo em questão, então bora hackear!

Tendo isso em mente, vou utilizar uma worldlist com diversos nomes de usuarios para realizar a enumeração

Então vamos continuar o nosso ataque.

```sh
kerbrute userenum --dc attacktive.thm -d spookysec.local userlist.txt | tee kerbrute.txt
```

![Kerbrute](./images/walkthroughs/attacktive-directory/img.attacktive-5.png)

Notamos que a ferramenta nos trouxe diversos usuarios validos e um hash do usuario svc-admin, esse usuario não necessita se autenticar atraves do kerberos, sendo uma falha muito grave de segurança ainda mais para um "admin"

Vamos quebrar o hash dele, mas antes precisamos pesquisar qual é o tipo do hash em questão, para isso vou utilizar a WiKi do Hashcat https://hashcat.net/wiki/doku.php?id=example_hashes e pesquisar na pagina o SALT do hash "$krb5asrep$" após isso validamos que o ID do hash (na ferramenta do hashcat) é o "18200" e o nome é "Kerberos 5 AS-REP etype 23"

![Hashcat-1](./images/walkthroughs/attacktive-directory/img.attacktive-6.png)

Após realizado a consulta, realizo o ataque de brute force

```sh
hashcat -m 18200 svc-admin.hash worldlist.txt
```
![Hashcat-2](./images/walkthroughs/attacktive-directory/img.attacktive-7.png)

E voalá! a senha do svc-admin é management2005

Agora vamos tentar nos conectar no seviço de SMB com as credenciais svc-admin:management2005 para procurar alguma configuração ou arquivo sensivel nas pastas compartilhadas.

```sh
smbclient -L //spookysec.local/ -U "svc-admin"
```

![Smbclient](./images/walkthroughs/attacktive-directory/img.attacktive-8.png)

Após fazer a enumeração localizo uma pasta chamada backup e dentro dela um arquivo chamado "backup_credentials.txt"
Possivelmente as credenciais do usuario backup que nós enumeramos à alguns passos atrás.

![Smbclient](./images/walkthroughs/attacktive-directory/img.attacktive-9.png)

O mesmo está emcriptado com base 64, como estudo sobre o assunto algum tempo, conheço alguns padrões de hash como o Base64

Então realizamos a conversão para texto claro com o comando:

```sh
echo -n "YmFja3VwQHNwb29reXNlYy5sb2NhbDpiYWNrdXAyNTE3ODYw" | base64 -d         
backup@spookysec.local:backup2517860  
```

Após fazer o decrypt, descobrimos as credenciais do usuario "backup" do dominio.
backup:backup2517860

Agora que conseguimos as credenciais do usuario backup, conseguimos realizar um dump dos hashes de todos os usuarios que estão sincronizados no AD, sendo assim conseguimos até as credenciais do sysadmin "administrator" pois o usuario backup, é responsavel pelo backup do DC.

Então partiu Hackear!

Para isso, vamos utilizar o secretsdump, uma ferramenta

que faz parte do impacket (https://github.com/SecureAuthCorp/impacket) que faz um dump (coleta) de todos os hashes remotamente (isso é jogo baixo!).

```sh
secretsdump.py spookysec.local/backup:backup2517860@spookysec.local
```

![Secretsdump](./images/walkthroughs/attacktive-directory/img.attacktive-10.png)

Agora que coletamos o hash do "administrator" temos duas opções, ou realizar a quebrar do hash NTLM utilizando alguma ferramenta de quebras de senha, como o hashcat, john.. 

Também podemos se conectar no servidor remotamente utilizando a tecnica "Pass The Hash", sendo assim, em vez de utilizar uma senha, iremos passar o hash para se autenticar no servidor, então não precisamos quebrar a senha do usuario, basta nos conectarmos e se quisermos, trocar a senha localmente

Vou utilizar a ferramenta chamada Evil-WinRM (https://github.com/Hackplayers/evil-winrm) que permite o uso da tecnica "Pass The Hash"

```sh
sudo /scripts/infra/evil-winrm/evil-winrm.rb -i spookysec.local -u administrator -H "0e0363213e37b94221497260b0bcb4fc"
```

![Shell-1](./images/walkthroughs/attacktive-directory/img.attacktive-11.png)

Agora que ganhamos uma shell no servidor, temos acesso completo a ele, então irei habilitar o RDP e ganhar um acesso grafico ao servidor.

Primeiro irei, desativar o firewall, depois habilitar o RDP e alterar a senha do "administrator".

```sh
netsh advfirewall set  currentprofile state off
```

```sh
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

```sh
net user administrator Pow3n3d_@@
```

![Shell-2](./images/walkthroughs/attacktive-directory/img.attacktive-12.png)

Feito isso, irei me conecar com o xfreerdp e conseguir acesso completo ao main server :)

```sh
xfreerdp /u:administrator /v:10.10.60.231
```

![RDP](./images/walkthroughs/attacktive-directory/img.attacktive-13.png)

***Espero que tenham gostado!***

***Partiu Hackear!***
