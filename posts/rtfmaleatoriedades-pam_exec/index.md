# RTFM&amp; a Leatoriedades - PAM_exec


# PAM - módulo pam_exec para execução de comando externo

Estou participando de um projeto, onde sou o ponto focal para Infra, de uma equipe composta por profissionais de países e empresas diferentes! E fui abordado por um Dev, com o seguinte pedido:

&gt; *Você consegue criar uma forma de, eu mesmo, fazer o restart da instância secundária do PGSQL do ambiente stage ? Sem me dar acesso ao servidor Linux para rodar um comando! E que seja simples para não gerar uma demanda de projeto para Infra etc...*

- *Naturalmente: perguntei o motivo! E prefiro não entrar na questão...*

Em detrimento do projeto e ambiente: minha diretiva principal também é apoiar com soluções que resolvam bloqueios da equipe!

Dito isso! Veio de imediato em minha mente:

- posso criar um conta de sistema! Dessa forma não há um shell válido para login
  - com diretório home apenas para o subdiretório \~/.ssh, para armazenar o authorized_keys com a chaves públicas
    - e numa tentativa de login via ssh, onde a chave pública for validada: um `systemctl restart ...` é então executado.
- seguido a ideia, de usar ssh, acho que é possível duas abordagens:
  - usando o módulo [pam_exec](https://man7.org/linux/man-pages/man8/pam_exec.8.html) do [PAM](https://man7.org/linux/man-pages/man8/PAM.8.html)! Configurado no arquivo de configuração PAM para o SSH: `/etc/pam.d/sshd`
  - usando o bloco condicional [Match](https://man7.org/linux/man-pages/man5/sshd_config.5.html#DESCRIPTION) do [SSHD](https://man7.org/linux/man-pages/man8/sshd.8.html) para usar o [ForceCommand](https://man7.org/linux/man-pages/man5/sshd_config.5.html)

Como há muito tempo não brinco com o PAM. Optei por ele!
- porém, deixo claro que: não me veio nenhuma outra ideia de como aplicar tecnicamente no Linux! Então, é um favor que você me faz, criticar o que proponho aqui! ;)

- Conta de sistema

Criando uma conta de sistema que será de uso compartilhado, caso algum outro Dev queira fazer uso também. Para tanto, criaremos também o diretório home para armazenar o subdiretório \~/.ssh para armazenar as chaves públicas SSH.

``` bash
useradd -r -s /usr/sbin/nologin -m -c &#39;user para restart pgsql&#39; restartpg &amp;&amp; getent passwd restartpg 
restartpg:x:995:986:user para restart pgsql:/home/restartpg:/usr/sbin/nologin
```

- /etc/ssh/sshd_config

1.  Porque não precaver...

``` bash
cp /etc/ssh/sshd_config{,.BKP}
```

2.  Garanta que o PAM esteja habilitado!  

``` bash
# Set this to &#39;yes&#39; to enable PAM authentication, account processing,
# and session processing. If this is enabled, PAM authentication will
# be allowed through the KbdInteractiveAuthentication and
# PasswordAuthentication.  Depending on your PAM configuration,
# PAM authentication via KbdInteractiveAuthentication may bypass
# the setting of &#34;PermitRootLogin yes
# If you just want the PAM account and session checks to run without
# PAM authentication, then enable this but set PasswordAuthentication
# and KbdInteractiveAuthentication to &#39;no&#39;.
UsePAM yes
```

- /etc/pam.d/sshd

1.  Não custa nada fazer um backup do arquivo ;)

``` bash
cp /etc/pam.d/sshd{,.BKP}
```

2.  Em seguida inserimos nossa configuração

``` bash
cat &lt;&lt;EoF &gt;&gt; /etc/pam.d/sshd 

# TCPIP ME - @faioli 10/09/24
# reboot postgresql by specific user
session    optional     pam_exec.so type=open_session debug log=/var/log/sshpamexec.sh.log seteuid /usr/local/bin/sshpamexec.sh
EoF
```

- /usr/local/bin/sshpamexec.sh

``` bash
  #!/usr/bin/env bash
  ##
  # Autor: Thiago Torres Faioli - A.K.A: 0xttfx - thiago@tcpip.net.br
  # Função: reiniciar instância PGSQL usando conta de sistema sem shell
  # válido que numa tentativa de login via ssh, onde a chave pública for
  # validada o comando `systemctl restart ...` é então executado.
  ##
  # Versão: 0.1
  # Data: 10 de Setembro de 2024
  # Licença: SPDX-License-Identifier: BSD-3-Clause
  #####################################################################
  
  # funcao para log
  _log_msg() {
    local msg=&#34;$1&#34;
    local tstamp=$(date &#43;&#34;%d-%m-%Y %H:%M:%S&#34;)
    echo &#34;[${tstamp}] ${msg}&#34; &gt;&gt; /var/log/sshpamexec.log
  }
  
  # funcão de restart e status do pgsql
  _restartpg (){
    /usr/bin/systemctl restart postgresql@16-main.service
    _log_msg &#34;$(/usr/bin/systemctl status postgresql@16-main.service --no-pager)&#34;
  }
  
  # variável com parser do fingerprint associado ao último login ssh do usuário restartpg 
  fprint=&#34;$(cat /var/log/auth.log |grep -iA1 &#34;accepted publickey&#34;|grep -B1 restartpg |grep -oE SHA.*$|/usr/bin/tail -n1)&#34;
  
  # varoável com arquivo temporário criado para armazenar lista de fingerprit
  # do arquivo authorized_keys do usuário restartpg 
  filelfp=&#34;$(mktemp /tmp/listakeyfp-XXX)&#34;
  # inserindo lista de fingerprints 
  for pubkey_file in /home/restartpg/.ssh/authorized_keys; do
    ssh-keygen -lf ${pubkey_file} -E sha256
  done &gt; ${filelfp} 
  
  # variárel com parser identificando o fingerprint
  loginfp=&#34;$(/usr/bin/grep ${fprint} ${filelfp}| /usr/bin/awk &#39;{print $3,$2}&#39;)&#34;
  
  _usr_parser (){
  if [ &#34;$PAM_USER&#34; = &#34;restartpg&#34; ]; then
    _log_msg &#34;- - - -&#34;
    _log_msg &#34;Usuário $PAM_USER reinicando instância PGSQL&#34;
    _log_msg &#34;Executado por: $loginfp&#34; 
    _log_msg &#34;-&#34;
    _restartpg
  else 
    _log_msg &#34;- - - -&#34;
    _log_msg &#34;Usuário $PAM_USER! Script $0 ignorado.&#34;
  fi
  } 
  _usr_parser
  rm -f $filelfp
  exit 0
```

- Testando

``` bash
$ ssh -l restartpg srvpgsql1.dns.com
Last login: Wed Sep 11 13:24:40 2024 from ???.??.???.???
This account is currently not available
Connection to srvpgsql1.dns.com exit closed.
```

- /var/log/sshpamexec.log

Esse será o log gerado das operações

``` bash
tail -30 /var/log/sshpamexec.log

[11-09-2024 11:12:32] - - - -
[11-09-2024 11:12:32] Usuário treta! Script /usr/local/bin/sshpamexec.sh ignorado.
*** Wed Sep 11 13:24:37 2024
[11-09-2024 13:24:37] - - - -
[11-09-2024 13:24:37] Usuário restartpg reinicando instância PGSQL
[11-09-2024 13:24:37] Executado por: ltorvalds@tcpip.net.br SHA256:nADfsdJgYyN8UFsfsdfdwer3rf3r33dwe3eiO&#43;QVwZE
[11-09-2024 13:24:37] -
[11-09-2024 13:24:40] ● postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: active (running) since Wed 2024-09-11 13:24:40 -03; 10ms ago
    Process: 38251 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=0/SUCCESS)
   Main PID: 38257 (postgres)
      Tasks: 6 (limit: 4657)
     Memory: 53.4M (peak: 62.3M)
        CPU: 303ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@16-main.service
             ├─38257 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
             ├─38258 &#34;postgres: 16/main: checkpointer &#34;
             ├─38259 &#34;postgres: 16/main: background writer &#34;
             ├─38261 &#34;postgres: 16/main: walwriter &#34;
             ├─38262 &#34;postgres: 16/main: autovacuum launcher &#34;
             └─38263 &#34;postgres: 16/main: logical replication launcher &#34;

Sep 11 13:24:37 stg-ccp-pgsql1 systemd[1]: Starting postgresql@16-main.service - PostgreSQL Cluster 16-main...
Sep 11 13:24:40 stg-ccp-pgsql1 systemd[1]: Started postgresql@16-main.service - PostgreSQL Cluster 16-main.
```


---
&lt;script src=&#34;https://giscus.app/client.js&#34;
        data-repo=&#34;0xttfx/0xttfx.github.io&#34;
        data-repo-id=&#34;R_kgDOK3wAHw&#34;
        data-category=&#34;BlogPostComments&#34;
        data-category-id=&#34;DIC_kwDOK3wAH84Cnmtb&#34;
        data-mapping=&#34;pathname&#34;
        data-strict=&#34;1&#34;
        data-reactions-enabled=&#34;1&#34;
        data-emit-metadata=&#34;0&#34;
        data-input-position=&#34;top&#34;
        data-theme=&#34;preferred_color_scheme&#34;
        data-lang=&#34;en&#34;
        data-loading=&#34;lazy&#34;
        crossorigin=&#34;anonymous&#34;
        async&gt;
&lt;/script&gt;



---

> Author: [Faioli a.k.a 0xttfx](https://github.com/0xttfx)  
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-pam_exec/  

