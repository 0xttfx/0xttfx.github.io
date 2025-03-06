# RTFM &amp; Aleatoriedades Para SysAdmin - Journalctl 


 Para quando a lei de Murphy exerce a sua força. É nessa hora, que do seu kit MacGyver, você tira o **journalctl**  para identificar o que pode estar errado!
 
 Das muitas formas possíveis. Tem mais essa:

```bash
$ journalctl --no-pager --since today --grep &#39;fail|error|fatal&#39; --output json|jq
```
- Com a opção &#34;--grep&#34; filtramos palavras chave
- Com a opção &#34;--since&#34; melhoramos nossa assertividade. 
	- Ex:
		- --since &#34;1 hour ago&#34;
		- --since &#34;1 minutes ago&#34;
		- --since &#34;5 seconds ago&#34;
		- --since 07:00 --until &#34;1 hour ago&#34;
- Com o &#34;--output&#34; usando o formato *json*, fazemos um parser amigável pra organizarmos com **jq**.

```bash
$ journalctl --no-pager --since today --grep &#39;fail|error|fatal&#39; --output json|jq

{
  &#34;_PID&#34;: &#34;5605&#34;,
  &#34;_SYSTEMD_SLICE&#34;: &#34;user-1000.slice&#34;,
  &#34;_SYSTEMD_USER_UNIT&#34;: &#34;org.gnome.Shell@wayland.service&#34;,
  &#34;_RUNTIME_SCOPE&#34;: &#34;system&#34;,
  &#34;_SYSTEMD_UNIT&#34;: &#34;user@1000.service&#34;,
  &#34;__SEQNUM&#34;: &#34;7540847&#34;,
  ...
  ...
  &#34;_SYSTEMD_OWNER_UID&#34;: &#34;1000&#34;,
  &#34;SYSLOG_IDENTIFIER&#34;: &#34;google-chrome.desktop&#34;,
  &#34;_COMM&#34;: &#34;cat&#34;,
  &#34;__MONOTONIC_TIMESTAMP&#34;: &#34;285995139402&#34;,
  &#34;_CMDLINE&#34;: &#34;cat&#34;,
  &#34;_BOOT_ID&#34;: &#34;da45ccab9c404e9444444a9c51300cae&#34;,
  &#34;_EXE&#34;: &#34;/usr/bin/cat&#34;
}
{
...
...
```

Ainda é possível filtrar por algum objeto com o *jq* e quantificar as ocorrências com sort e uniq
- Talvez, alguns objetos interessantes seriam _EXE, _CMDLINE, _PID, SYSLOG_IDENTIFIER, MESSAGE...    
```bash
$ journalctl --no-pager --since &#34;60 minutes ago&#34; --grep &#39;fail|error|fatal&#39; --output json|jq &#39;.SYSLOG_IDENTIFIER&#39; | sort | uniq -c
      1 &#34;audit&#34;
      1 &#34;cupsd&#34;
     19 &#34;discord.desktop&#34;
      2 &#34;fprintd&#34;
      7 &#34;gnome-shell&#34;
  10518 &#34;google-chrome.desktop&#34;
     18 null
      4 &#34;org.gnome.Software.desktop&#34;
    173 &#34;slack.desktop&#34;
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
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-journalctl/  

