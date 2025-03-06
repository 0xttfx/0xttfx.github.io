# RTFM &amp; Aleatoriedades - UDEV_Keychron


Para explorar as features do teclado V10 é utilizado o software [usevia.app](https://usevia.app) diretamente pelo navegador! E a parte boa é a sua compatibilidade com Linux.

- meu cenário:
  - Fedora release 40 (Forty)
  - Google Chrome
- Basta acessar o link do aplicativo [usevia.app](https://usevia.app) e clicar em **Authorize device**

![img1](/images/UDEV/useviaauthorize.png)

&gt;[!WARNING]
&gt;Porem ao tentar fazer a autorizacão do dispositivo! Ocorre o erro de permissão de acesso

![img2](/images/UDEV/useviaerror.png)

&gt;[!note]
&gt;é preciso criar uma regra `udev` para o driver Linux [hidraw](https://www.kernel.org/doc/Documentation/hid/hidraw.txt) que é utilizado para comunicacão com o teclado.

## Solucão

Primeiro liste as infos do barramento USB pra coletar os dados do device

- [lsusb](https://man7.org/linux/man-pages/man8/lsusb.8.html)
  ou
- [udevadm](https://man7.org/linux/man-pages/man8/udevadm.8.html)

#### Obtendo os dados

- **lsusb**
  Sem rodeios

``` bash
$ lsusb
...
...
Bus 001 Device 005: ID 3434:03a1 Keychron Keychron V10
...
```

&gt; \[!tip\]
&gt; [lsusb](https://man7.org/linux/man-pages/man8/lsusb.8.html#OPTIONS) -D /dev/bus/usb/001/005

- **udevadm**
  Com o [`udevadm`](https://man7.org/linux/man-pages/man8/udevadm.8.html) monitore o sistema [UDEV](https://mirrors.edge.kernel.org/pub/linux/utils/kernel/hotplug/udev/udev.html) [`monitor --property`](https://man7.org/linux/man-pages/man8/udevadm.8.html#OPTIONS)

1.  remova o teclado
2.  inicie o monitoramento
3.  conecte o teclado

``` bash
$ sudo udevadm monitor -up |grep -E &#34;ID_VENDOR_ID|ID_MODEL_ID|DEVNAME&#34;
...
DEVNAME=/dev/bus/usb/001/005
ID_MODEL_ID=03a1
ID_VENDOR_ID=3434
```

Com o path do device em mãos, também podemos obter dados com o comando [`udevadm`](https://man7.org/linux/man-pages/man8/udevadm.8.html) usando a opcão [`info`](https://man7.org/linux/man-pages/man8/udevadm.8.html#OPTIONS)

- nos interessa os campos:
  - ID_MODEL_ID
  - ID_VENDOR_ID

&gt; \[!tip\]
&gt; \$ sudo udevadm info /dev/bus/usb/001/005 \|grep -E &#34;ID_VENDOR_ID\|ID_MODEL_ID&#34;

Sabemos que:
- o barramento é: **001**
- o dispositivo é: **005**
- o ID do vendor Keychron é: **3434**
- o ID do produto Keychron V10 é: **03a1**

## Regra UDEV

Específica para o device [hidraw](https://www.kernel.org/doc/Documentation/hid/hidraw.txt)

- primeiro crie uma variável com o group id do usuário
  - meu usuário é `0xttfx`

  ``` bash
  export USER_GID=`id -g 0xttfx`
  ```
- enfim a regra:

``` bash
$ sudo --preserve-env=USER_GID sh -c &#39;cat &lt;&lt;EOF &gt; /etc/udev/rules.d/via.rules
# Keychron_V10
KERNEL==&#34;hidraw*&#34;, SUBSYSTEM==&#34;hidraw&#34;, ATTRS{idVendor}==&#34;3434&#34;, ATTRS{idProduct}==&#34;03a1&#34;, MODE=&#34;0660&#34;, GROUP=&#34;${USER_GID}&#34;, TAG&#43;=&#34;uaccess&#34;, TAG&#43;=&#34;udev-acl&#34;
EOF&#39;
```

Em seguida recarregue as regras UDEV

``` bash
sudo sh -c &#39;udevadm control --reload-rules &amp;&amp; udevadm trigger&#39;
```

Funfando :)
![img3](/images/UDEV/useviaok.png)

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
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-udev_keychron/  

