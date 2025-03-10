# RTFM &amp; Aleatoriedades - Swap


Swapping foi introduzida como um backup em disco para páginas não mapeadas. E existem três tipos de páginas que devem ser tratadas pelo subsistema de swapping:

- Pages que pertencem a uma região de memória anônima de um processo (User Mode stack)
- Dirty pages  que pertencem a um mapeamento de memória privada de um processo
- Pages que pertencem a uma região de memória compartilhada IPC

Numa &#34;Regular Paging&#34; cada entrada na &#34;Page Table&#34; inclui um sinalizador, uma &#34;Present flag&#34; e o Kernel explora esse sinalizador para sinalizar que uma página pertencente a um espaço de endereço de um processo qualquer foi &#34;swapped out&#34;! E além desse sinalizador, o Linux faz uso dos bits restantes do &#34;Page Table&#34; pra armazenar um identificador de &#34;swapped-out page&#34; que informa a localização no disco. 

A Principais características do subsistema swapping  são:

- Configura area swap para armazenamento de Pages que não possuem &#34;disk image&#34;.
- Gerencia espaço na área de swap alocando e liberando &#34;page slots&#34;.
- Fornece a função de &#34;swap out&#34; pages da RAM para a área de swap  e &#34;swap in&#34; pages da área de swap para RAM.
- Faz uso da “swapped-out page identifiers” das entradas na &#34;Page Table&#34; que foram swapped para acompanhar as posições dos dados na área de swap.

Em suma, o swapping é o principal recurso de &#34;page frame reclaiming&#34;! E se queremos ter certeza que todos os &#34;page frames&#34; obtidos por um processo, não apenas os &#34;pages&#34; que contem &#34;disk image&#34;, possam ser recuperados pelo PFRA, devemos fazer uso do swapping...

Com isso podemos deduzir que grandes áreas de swap dão poder ao Kernel para iniciar vários processos onde o total de solicitações de memória ultrapassa a quantidade de RAM física.

E como na TI, nem tudo são flores, precisamos nos atentar que simular RAM em disco, nos traz um desempenho em milissegundos se comparado aos nanosegundos da RAM física ;) 

&gt;[!NOTE]
&gt;*Referencia: Daniel P. Bovet, Marco Cesati - Understanding the Linux Kernel - Capítulo 2 - Memory Addressing e Capítulo 17 - Page Frame Reclaiming*


Agora que nivelamos a introdução sobre swapping, vamos criar um espaço de troca, do tipo arquivo, que não muda em nada quando partição em disco: a não ser o processo de particionamento de disco etc...


1. Verificando se há algum swap

```bash
sudo swapon --show
```

2. Criando arquivo de swap de 2GiB

```bash
sudo dd if=/dev/zero of=/swapfile count=2192 bs=1MiB
```

3. Alterando permissão do arquivo

```bash
sudo chmod 600 /swapfile
```

4. Marcando o arquivo como um espaço de swap

```bash
sudo mkswap /swapfile
```

5. Verificando se está tudo ok

```bash
sudo swapon --show
```

6. Adicionando a linha no &#39;/etc/fstab&#39; para montagem automática

```bash
echo &#39;/swapfile    none    swap    sw    0 0&#39; | sudo tee -a /etc/fstab
```

7. Ajustando a configuração de Swap

O parâmetro `/proc/sys/vmwappiness` configura a frequência com que o sistema transfere dados da RAM para o espaço de swap. Sendo um  valor entre 0 e 100 que representa uma porcentagem! Um valor baixo significa que o seu sistema Linux troca processos raramente enquanto um alto valor significa que os processos são gravados em disco imediatamente

* Com valores próximos de zero, o kernel não irá transferir dados para o  disco a menos que seja absolutamente necessário.
* Lembre-se, as  interações com o arquivo de swap são “dispendiosas”! Pois são mais lentas que as interações com a RAM.
* Valores que estão mais próximos de 100 irão tentar colocar mais dados  no swap em um esforço para manter mais espaço da RAM livre.
	* Dependendo  do perfil de memória de seus aplicativos ou do motivo pelo qual você  está usando o seu servidor, isso pode ser melhor em alguns casos.

|     |     |
| --- | --- |
| Swappiness: 60 | Swap a partir de 40% de uso de RAM. |
| Swappiness: 40 | Swap a partir de 60% de uso de RAM. |
| Swappiness: 20 | Swap a partir de 80% de uso de RAM. |
| Swappiness: 10. | Swap a partir de 90% de uso de RAM. |
| Swappiness: 1 | Swap a partir de 99% de uso de RAM. |

Para um desktop, um valor de swappiness de 60 não é um valor ruim e normalmente é o valor dfault em uma distro Linux. Mas para um servidor, podemos deixá-lo mais próximo de 0, para fazer uso somente quando realmente necessário

Como exemplo podemos setar para  &#39;10&#39; para que o swap seja utilizado após 90% de RAM ocupada
```bash
sudo sysctl vm.swappiness=10
```
Para garantir que esse valor irá se manter após um boot
```bash
echo &#39;vm.swappiness=10&#39; | sudo tee -a /etc/sysctl.conf
```

Mas como sempre, não há uma receita de bolo! Em determinados cenários  trocar processos de tempo de execução da RAM para o disco, deve ser evitado, noutros há vantagem... Conhecer a aplicação irá lhe guiar na configuração mais apropriada!

8. Ajustando a confiuração do vfs\_cache\_pressure

Esta opção controla a tendência do kernel de recuperar a memória que é usada para cache de diretórios e objetos inode.

* No valor padrão de vfs\_cache\_pressure=100, o kernel tentará recuperar dentries e inodes a uma taxa &#34;justa&#34; em relação ao pagecache e à recuperação do swapcache.
* Diminuir vfs\_cache\_pressure faz com que o kernel prefira manter os caches dentry e inode.
* Quando vfs\_cache\_pressure=0, o kernel nunca recuperará dentries e inodes devido à pressão de memória e isso pode levar facilmente a condições de falta de memória.
* Aumentar vfs\_cache\_pressure além de 100 faz com que o kernel prefira recuperar dentries e inodes.

Podemos definir isso em um valor mais conservador como 50
```bash
sudo sysctl vm.vfs_cache_pressure=50
```

Vamos garantir que após um boot o valor permaneça
```bash
echo &#39;vm.vfs_cache_pressure=50&#39; | sudo tee -a /etc/sysctl.conf
```

9. Agora sim! Vamos ativar o novo espaço de swapping 

```bash
sudo swapon /swapfile
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
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-swap/  

