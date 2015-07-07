### Adicionando Máquinas Linux ao Domínio - Active Directory 2012.

O avahi utiliza os domínios .local e ponto .foo como reservados apenas para serviços da maquina local, ou seja dentro do próprio computador, então para burlamos esta questão teremos que modificar o arquivo **/etc/nsswitch.conf**.

Esta linha deverá ser modificada:

```sh
hosts:          files  mdns4_minimal [NOTFOUND=return] dns
```
Para esta:


```sh
hosts:          files dns mdns4_minimal [NOTFOUND=return]
```

Assim as consultas serão feitas 1º no dns especificado depois no serviços avahi.

Faça um teste verificando o acesso ao servidores via ping.

```bash
$ ping ifto.local
$ ping dc001.ifto.local
$ ping dc002.ifto.local
$ ping palmas.ifto.local
```
### PACOTES NECESSÁRIOS PARA A INSTALAÇÃO  DA AUTENTICAÇÃO

```bash
$ sudo apt-get -y install realmd sssd sssd-tools samba-common krb5-user packagekit samba-common-bin samba-libs adcli ntp
```
Nesta etapa o cliente Kerberos ira solicitar algumas informações:

```sh
IFTO.LOCAL
```

![krb5-user-config](/figuras/01)


**SERVIDORES DC**

**Obs: coloque os nomes dos mesmos separados por espaços**

```sh
DC001.IFTO.LOCAL DC002.IFTO.LOCAL PALMAS.IFTO.LOCAL
```
![krb5-user-config](/figuras/02)


**POR ULTIMO O SERVIDOR PRINCIPAL DO DOMINIO O QUAL POSSUI O CONTROLE DAS CHAVES**

```sh
DC001.IFTO.LOCAL
```

![krb5-user-config](/figuras/03)



Os clientes do Active Directory devem sempre estar com o horário atualizado assim
vamos obrigar o cliente a sincronizar o seu horário local com os dos servidores AD
locais.

Faça alterações no arquivo **/etc/ntp.conf** conforme a figura abaixo:

![NTP CONFIG](figuras/04)

```sh
server dc001.ifto.local
server dc002.ifto.local
server palmas.ifto.local
```
**Obs.:** Não esqueça de commentar as demais linhas que contém o parametro **server**

**Restart o serviço ntp**

```bash
$ sudo service ntp restart
```

###Configurar o Realmd - (Cliente Kerberos)

Este serviço fara a integração entre cliente e servidor.

***Obs:*** Muito cuidado nesta parte da configuração.

Crie um novo arquivo em /etc/realmd.conf com os seguintes paramentros:

[users]

default-home = /home/%D/%U

default-shell = /bin/bash

[active-directory]

default-client = sssd

os-name = Ubuntu Desktop Linux

os-version = 14.04

[service]

automatic-install = no

[ifto.local]

fully-qualified-names = no

automatic-id-mapping = yes

user-principal = yes

manage-system = no


###Adicionando maquina ao Dominio

* 1º Ativar o ticket do Kerberos:

```bash
$ sudo kinit administrator@IFTO.LOCAL
```

**Digite sua senha e pronto.**

* 2º Comando para adicionar a maquina no AD.

```bash
$ sudo realm --verbose join ifto.local -U administrador
```

* 3º Verificando se nossos passos deram certo

```bash
$ sudo realm list

```
![NTP CONFIG](figuras/06)

* 4º Permitir que qualquer usuário do dominio faça login do computador:

```bash
$ sudo realm permit --all

```
* 5º Alterando configuração do serviço sssd (System Security Services Daemon)

**Obs:** O realmd configura em partes o sssd mais existe um problema com arquivo vamos corrigir isso agora.

Alterando a linha dentro do arquivo **/etc/sssd/sssd.conf**:

```sh
access_provider = simple
```

Para:

```sh
access_provider = ad
```
**Reiniciando o serviço**

```bash
$ sudo service sssd restart.
```

**Opção Extra**

Caso voce queira refinar as configurações do sssd para que a maquina cliente por exemplo tente acessar o AD com 3 tentativas, adicione estas linhas abaixo dentro do arquivo: **/etc/sssd/sssd.conf**.

```sh
[nss]
filter_groups = root
filter_users = root
reconnection_retries = 3

[pam]
reconnection_retries = 3
```

Ficando desta forma o arquivo:

![SSSD CONFIG](figuras/07)


**Caso você efetue as alterações da opção Extra do 5º passo reinicie o serviço.**

```bash
$ sudo service sssd restart.
```




* 6º Configurar a criação automática do diretório home no login do usuário

Dentro do arquivo: **/etc/pam.d/common-session**
* Adicione a linha -

```sh
session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
```
Ficando desta forma:

![NTP CONFIG](figuras/05)

* 7º Adicionar permissões para grupos específicos possam por exemplo instalar programas via sudo.

Edite o arquivo **/etc/sudoers**

Adicione esta linha:

```sh
%g_admins_ad ALL=(ALL) ALL
```

** PRONTO REINICE O COMPUTADOR E TENTE EFETUAR O LOGIN **



## EXTRAS

No ubuntu 14.04 para que os usuários não vejam a lista de logins efetuados na tela principal de login, crie o arquivo: **/etc/ligthgdm/ligthgdm.conf**
com as seguintes linhas:

```sh
[SeatDefaults]
allow-guest=false
greeter-hide-users=true
greeter-show-manual-login=true
```

### SOLUÇÃO IMPLEMENTADA NO IFTO - CAMPUS PALMAS
Dúvidas e Sugestões

CONTATO: HUGO CAVALCANTE LIMA

EMAIL: hugo@ifto.edu.br


Abraços  :-)
