# IP DOK Set or Drop Sec Group AWS EC2 RDS


![DO-AWS](/images/IPDO/docean_aws.png)

# ipdo-awsrds

## Função
 Scrip bash para descoberta dos IPs WAN dos DOcean Droplets e inserção em security groups EC2 das instâncias AWS RDS.

## Atualizações
 
- v0.4
  - Como inteválo mínimo de execução na CRON são de 60 segundos! E eu precisava executar por mais vezes por minuto
    acabei contendo o scrip dentro de um laço wile que o executa por 10 vezes com intervalo de 2 segundos 
- v0.5
  - removido laço de x execuções a cada 60s
  - restruturado algumas conditional statement da inserção e remoção de IPs
  - O diff, antes realizado globalmente, agora é realizado para cada Security Group
- v0.6
  - Implementado o builtin *set* com as *options* *errtrace, errexit, nounset e pipefail* para deixar o script mais criterioso quanto a erros Principalmente os de APIs 5xx sem a necessidade de fazer testes em funções, condições etc.
  - Implementado trap para melhor localização do momento e local do erro.
- v0.7
  - Implementado arquivo PID.
  - Modificado método de criação de array&#39;s específicos para uso mapfile
  - Melhorias no filtro diff dos arquivos de IPs
  - Melhorias nos filtros para criação de array IPs
  - Modificado no laço aninhado de inserção e remoção de IP para melhora do parser  
- v0.8
  - Implementado tratativa para cod error diff
  - Implementado arquivo de diff temporário para cada sec. group verificado
  - Implementado tratativa para cod error na criação de array usando arquivo diff temporário
  - Implementado remoção de arquivos temporários em cada laço
- v1.0
  - Agora sim! **ipdo-awsrds** agora é GA - Disponibilidade Geral!
  - Versão estável para uso em ambiente produção. 

## Dependências

- AWS CLI
  - [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  - [EC2](https://docs.aws.amazon.com/cli/latest/reference/ec2/)
    - [API authorize-security-group-ingress](https://docs.aws.amazon.com/cli/latest/reference/ec2/authorize-security-group-ingress.html)
    - [API describe-security-groups](https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-security-groups.html)
  - [RDS](https://docs.aws.amazon.com/cli/latest/reference/rds/)
    - [API describe-db-instances](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-instances.html)
- DigitalOcean CLI
  - [DO CLI](https://docs.digitalocean.com/reference/doctl/how-to/install/)
    - [API compute](https://docs.digitalocean.com/reference/doctl/reference/compute/)

## Executando

Crie diretório `tools` e `tool/log` em  `/usr/local` 
```bash
mkdir -p /usr/local/tools/log &amp;&amp; cd /usr/local/tools/ &amp;&amp; \
git clone git@github.com:0xttfx/ipdo-awsrds.git
```

Agora criamos o arquivo *path-tools.sh* em */etc/profile.d/* para configurar o PATH para diretório *tools* 
```bash
sudo &gt; /etc/profile.d/path-tools.sh
```

Em seguida adicione o conteúdo:
```bash
# configurando PATH para incluir diretório tools caso exista.
if [ -d &#34;/usr/local/tools/ipdo-awsrds&#34; ] ; then
    PATH=&#34;$PATH:/usr/local/tools/ipdo-awsrds&#34;
fi
Eof
```

Para execução do script é necessário declarar o *user profile aws cli* usando a opção **-u**
```bash
ipdo-awsrds -u &lt;nome&gt;
```

## Automação 
 Para automação da execução, adicione a seguinte linha na crontab do usuário(*crontab -e*) não privilégiado.
 Devido ao update, adição e remoção dos nós dos clusters, o script será executado a cada 1 mintuo!
 - altere conforme sua necessidade.
```bash
* * * * *    /usr/bin/bash -x /usr/local/tools/ipdo-awsrds -u nome &gt;&gt; /usr/local/tools/log/ipdo-awsrds-$(date --date=&#34;today&#34; &#43;\%d\%m\%Y_\%H\%M\%S).log 2&gt;&amp;1
0 0 * * *    /usr/bin/find /usr/local/tools/log/ -type f -mtime &#43;5 -name &#39;exec-*.log&#39; -exec rm {} &#43;
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

> Author: 0xttfx  
> URL: http://0xttfx.tcpip.net.br/posts/ipdo-awsrds/  

