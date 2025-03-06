# RTFM &amp; Aleatoriedades - Windows Server Troubleshooting


Aqui temos uma abordagem simples de identificação de causa raiz para troubleshooting em ambientes Microsoft Windows Server.

Como exemplo estou usando um problema, que a equipe Microsoft de uma multinacional, enfrentou com um produto de mercado que estava causando um alto uso de Non-Paged Pool Memory(memory leak). E que causou, durante uma rotina de madrugada, o travamento de &#43;500 servidores...

- eu era de uma equipe multidiciplinar. responsável por virtualização, storage, backup, Unix e Linux e acabei me envolvendo pois as equipes de Segurança e Microsoft estavam apenas trocando acusações e ainda não identificado a causa raiz... 

## Problema
Verificado que o *non-paged pool* está com tamanho anormal

![unknown_filename.2.png](/images/WindowsServerTroubleshooting/unknown_filename.2.png)

- Conforme documentação da Microsoft, o Non-paged pool, pode ocupar até 75% da memória física. 
	- Porém não deve ser considerado como &#34;normal&#34; o grande uso da mesma, principalmente sem uma investigação etc...

- O Non-paged Pool são dados na RAM do computador usados ​​pelo kernel e pelos drivers do sistema operacional. 
	- Non-paged pool nunca é enviado para o disco ([Page Cache](https://0xttfx.github.io/posts/pagecache/))!  Sempre ficará armazenado na memória física.

- Conforme a minha vivência, leak de memória, é quando um programa que armazena dados no pool de memória não paginável do sistema faz &#34;merda&#34;. Simples assim!

## Investigação
Identificando o que está utilizando os *Pools Memory* no servidor Windão.

- Para o troubleshooting de Pool Memory é usada a ferramenta [PoolMon](https://learn.microsoft.com/pt-br/windows-hardware/drivers/devtest/poolmon) que é um monitor que exibe dados que o Windão coleta sobre alocações de memória dos pools de kernel paginados e não paginados do sistema... 
     - Os dados são agrupados por marca de alocação de pool que evidencia as tags que identificam o software etc... 

Chega de conversa! Para tanto abra o **command**  ou **power shell** e navegue até a pasta em que o executável &#34;poolmon.exe&#34;  está localizado:

- Pode-se executar a ferramenta para que seja gerado um log:

```powershell
C:\Temp\Windows Kits\10\Tools\x64&gt; poolmon -b -n C:\Temp\log_poolmon.txt
```

- Ou imprimir na STDOUT:

```powershell
C:\Temp\Windows Kits\10\Tools\x64&gt; poolmon -b
```
- “-b” executa o aplicativo e organiza os processos em ordem de consumo.

![unknown_filename.png](/images/WindowsServerTroubleshooting/unknown_filename.png)

Aqui conseguimos ver que no topo da lista, temos o processo com maior consumo de RAM &#34;**5053046496**&#34; Bytes e sua Tag é &#34;**NFeS**&#34;.   
- a TAG é um valor único e que é dado pela fabricante do driver.
```text
5053046496 / 1024 = 4.934.615,... / 1024 = 4.818,... / 1024 = 4,7...GB
```

 Com a tag em mãos, deve-se usar a ferramenta [**findstr**](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/findstr) que procura por cadeia de caracteres! Dessa forma vamos tentar localizar o arquivo que possui a tag em seu conteúdo...

- Vamos em seguida acessar o diretório de drives no Windows:
    - **path:** c:\windows\system32\drivers

```powershell
C:\&gt; cd c:\windows\system32\drivers
```
- E executar a tool para a busca da tag
```powershell
C:\Windows\System32\drivers\&gt; findstr /m /l /s NFeS *.sys
C:\Windows\System32\drivers\mfeavfk.sys
```
- /l: pesquisa por caracteres literais
- /m: lista apenas resultados correspondentes
- /s: pesquisa recursiva

Bingo....\\o/! 
- Identificamos com sucesso um aquivo que possui a tag: &#34;**mfeavfk.sys**&#34;!

Agora para tentar identificar o programa que está usando esse driver. 
- Vamos usar a ferramenta [**sigcheck**](https://learn.microsoft.com/pt-br/sysinternals/downloads/sigcheck)  que mostra metadados como, por exemplo: o número da versão do arquivo, carimbo de data/hora e detalhes da assinatura digital etc...

```text
C:\&gt; sigcheck c:\Windows\System32\drivers\mfeavfk.sys
```

![unknown_filename.1.png](/images/WindowsServerTroubleshooting/unknown_filename.1.png)

Assim identificamos que o proprietário é o **Anti-vírus McAfee**.

## Sobre o Arquivo e Driver Identificado

- Dados do Host afetado
	- Host: Windows Server
	- Versão: Windows Server Enterprise 2012 R2 x64

 - Versões do produto MCAfee:
	 - McAfee Agent - version: 5.6.2.209
	 - McAfee Endpoint Security Plataform - version: 10.6.11910
	 - McAfee Endpoint Security Threat Prevention - version: 10.6.11910

### Informações relevantes
Dos processos iniciados e drivers utilizados pelo Virus Scan Enterprise no Windão:

- O driver **mfeavfk**:
    - é um filtro para verificação de arquivos de sistema e para manter um cache desses arquivos aferidos.
  
- O arquivo **mfeavfk.sys**:
	- é um componente de software do **SYSCORE** da McAfee que opera como **Anti-Virus Filter Link** para os produtos de segurança do McAfee. 
		- O **VSE**( VirusScan Enterprise) através do driver **mfeavfk** que opera a nível de kernel, juntamente com o **Microsoft Filter Manager**, consegue verificar o vírus em tempo real e de forma autônoma quando arquivos são abertos ou fechados.
		    - Basicamente é assim que a McAfee identifica um vírus manipulando arquivos etc...
	- **MFEAVFK** é um acrônimo para = McAfee For Enterprise Anti-Virus File Link Filter Driver

- Características
	- Mfeavfk.sys não é essencial para SO e por não ser nativo do sistema causa relativamente poucos problemas.
	- Mfeavfk.sys está localizado na pasta C:\\Windows\\System32\\drivers
	- O tamanho do arquivo no Windão tem em média 91.672 bytes.
	- O driver pode ser iniciado ou parado quando feito de forma canônica.
	- O programa não está visível no task manager.
	- É assinado pela Verisign.
	- Não há descrição detalhada do serviço.
	- mfeavfk.sys parece ser um arquivo compactado. Portanto, a classificação de segurança técnica é 0% perigosa;
		- **no entanto**: ao pesquisar reviews da comunidade a quantidade de problemas relacionados são incontáveis... :(


Pesquisando na documentação técnica do produto, é informado pelo fabricante, que programas de backup e criptografia também usam o mesmo mecanismo:
- Por isso, conflitos e até mesmo corrompimentos de arquivos, podem ocorrer. 
- Até mesmo quando um vírus, que possui mecanismo de proteção contra o driver &#34;mfeavfk&#34;, pode gerar problemas críticos ao tentar matar o processo que faz uso do driver etc... 

Continuando com a pesquisa, mas agora, tentando cruzar ocorrências do driver &#34;mfeavfk.sys&#34; e leaks de &#34;Non-Paged Pool&#34; na base de conhecimento McAfee e Internet:
- O primeiro matching foi certeiro!
	- Encontrei diversos casos conhecidos pela McAfee do problema que estamos enfrentando e justamente com a mesma versão de SO e Anti-Vírus 😉
	- Segue link onde o assunto é extensivamente tratado entre clientes e suporte McAfee
		- [Link original](&lt;https://community.mcafee.com/t5/Endpoint-Security-ENS/Nonpaged-pool-memory-leak-mfeavfk-sys-on-multiple-windows-server/td-p/617169&gt;) está quebrado! Egraçado como as fabricantes desaparecem com alguns dados ... 
			- Mas a [WayBackMachine](https://web.archive.org/web/20201028165000/https://community.mcafee.com/t5/Endpoint-Security-ENS/Nonpaged-pool-memory-leak-mfeavfk-sys-on-multiple-windows-server/td-p/617169) salvou uma cópia para a posteridade 😉

![unknown_filename.3.png](/images/WindowsServerTroubleshooting/unknown_filename.3.png)

Logo em seguida, repassei essa investigação para as equipes de SegInfo e Microfofy para continuidade do troubleshooting...


**Referências**

&lt;https://kc.mcafee.com/corporate/index?page=content&amp;id=KB65784&gt;
&lt;https://www.file.net/process/mfeavfk.sys.html&gt;
&lt;https://support.microsoft.com/pt-br/help/177415/how-to-use-memory-pool-monitor-poolmon-exe-to-troubleshoot-kernel-mode&gt;
&lt;https://developer.microsoft.com/pt-br/windows/hardware&gt;
&lt;https://docs.microsoft.com/en-us/sysinternals/downloads/sigcheck&gt;
&lt;https://community.mcafee.com/t5/Endpoint-Security-ENS/Nonpaged-pool-memory-leak-mfeavfk-sys-on-multiple-windows-server/td-p/617169&gt;
&lt;https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filter-manager-concepts&gt;

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
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-windowsservertroubleshooting/  

