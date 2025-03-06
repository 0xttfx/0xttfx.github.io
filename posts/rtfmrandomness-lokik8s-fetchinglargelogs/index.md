# RTFM &amp; Randomness Loki K8s - Fetching Large Logs


There are several reasons why Grafana may not be configured to break the 5000 line limit when exploring logs... It is also known that we can export in several other ways. But here I will focus on solving the need to download an unexpectedly large amount of logs...

And for that we will make use of LogCLI: the command line interface for Grafana Loki that makes it easy to run [LogQL](https://grafana.com/docs/loki/latest/query/) queries on a Loki instance.

### Download binary

- Download the `logcli` binary from the [Loki releases page](https://github.com/grafana/loki/releases).
  - [loki-3.3.0.x86_64.rpm](https://github.com/grafana/loki/releases/download/v3.3.0/loki-3.3.0.x86_64.rpm)
  - [logcli_3.3.0_arm64.deb](https://github.com/grafana/loki/releases/download/v3.3.0/logcli_3.3.0_arm64.deb)

### Install

``` bash
sudo dnf isntall https://github.com/grafana/loki/releases/download/v3.3.0/logcli-3.3.0.x86_64.rpm
```

### Bash completion

add this to your `~/.bashrc.d` file

``` bash
cat &lt;&lt; EoF &gt;&gt; ~/.bashrc.d/var_loki
# Set up command completion
eval &#34;$(logcli --completion-script-bash)&#34;
EoF
```

If you have questions about the [query](https://grafana.com/docs/loki/latest/query/) that needs to be made, access the Explorer in Grafana

1.  Go to `Grafana` \&gt; `Explore`
    1.  Apply the necessary `label` to filter logs ...

![architecture](/images/Loki/lokifetching.png)

In this example, I&#39;m looking for the last 7 days of logs from the NGINX Ingress which is processing approximately 8.8 GiB.

``` bash
{app=&#34;ingress-nginx&#34;} |= ``
```

Then, to allow local access, forward a local port to the Loki pod

``` bash
kubectl --namespace loki port-forward svc/loki-gateway 8080:80
```

Configure the LogCli environment variable

``` bash
export LOKI_ADDR=http://localhost:8080
```

And using the query we checked, we can finally extract the log to a local file

``` bahs
$ logcli query &#39;{app=&#34;ingress-nginx&#34;} |= ``&#39; --limit=9000000 --since=168h  -o default,raw or jsonl  &gt; /tmp/log_ingres-nginx.log
```

- `--limit` here I try to set a very high value to ensure that all logs from the 7 days are captured.
- `--since` I set 168h time range.


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
> URL: http://0xttfx.tcpip.net.br/posts/rtfmrandomness-lokik8s-fetchinglargelogs/  

