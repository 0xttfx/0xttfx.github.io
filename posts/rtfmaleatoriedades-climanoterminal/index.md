# RTFM &amp; Aleatoriedades - Clima No Terminal


![climaNoTerminal](/images/ClimaCMD/climaNoTerminal.png)


O [wttr.in](https://github.com/chubin/wttr.in) é um serviço de previsão meteorológica para console que suporta vários métodos de representação de informações, como códigos de escape ANSI orientados a terminais para clientes HTTP de console (curl, httpie ou wget), HTML para navegadores web ou PNG para visualizadores gráficos.

Não tem segredo! Basta usar uma tool como [Client URL - curl](https://curl.se)  para buscar o conteúdo da URL `https://wttr.in`

- Seguindo esse exemplo, a página será renderizada já com os dados de geolocalização do seu IP e a previsão para o dia corrente e os próximos 2:
```bash
$ curl https://wttr.in 

Weather report: Belo Horizonte, Brazil

   _`/&#34;&#34;.-.     Shower in vicinity, thunderstorm in vicinity
    ,\_(   ).   &#43;27(29) °C     
     /(___(__)  → 12 km/h      
      ⚡‘‘⚡‘‘  10 km          
      ‘ ‘ ‘ ‘   0.4 mm         
                                                       ┌─────────────┐                                                       
┌──────────────────────────────┬───────────────────────┤  Wed 22 Jan ├───────────────────────┬──────────────────────────────┐
│            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
│     \   /     Sunny          │     \   /     Sunny          │  _`/&#34;&#34;.-.     Patchy light r…│  _`/&#34;&#34;.-.     Light rain sho…│
│      .-.      &#43;24(26) °C     │      .-.      &#43;30(32) °C     │   ,\_(   ).   &#43;27(30) °C     │   ,\_(   ).   19 °C          │
│   ― (   ) ―   ↖ 3 km/h       │   ― (   ) ―   ↓ 3 km/h       │    /(___(__)  ↗ 5-8 km/h     │    /(___(__)  ↖ 15-33 km/h   │
│      `-’      10 km          │      `-’      10 km          │     ⚡‘‘⚡‘‘  10 km          │      ‘ ‘ ‘ ‘  10 km            │
│     /   \     0.0 mm | 0%    │     /   \     0.0 mm | 0%    │     ‘ ‘ ‘ ‘   1.4 mm | 100%  │     ‘ ‘ ‘ ‘   1.3 mm | 100%  │
└──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                       ┌─────────────┐                                                       
┌──────────────────────────────┬───────────────────────┤  Thu 23 Jan ├───────────────────────┬──────────────────────────────┐
│            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
│     \   /     Sunny          │  _`/&#34;&#34;.-.     Thundery outbr…│  _`/&#34;&#34;.-.     Patchy light r…│     \   /     Clear          │
│      .-.      &#43;23(25) °C     │   ,\_(   ).   &#43;28(30) °C     │   ,\_(   ).   &#43;28(31) °C     │      .-.      &#43;23(25) °C     │
│   ― (   ) ―   ← 6-8 km/h     │    /(___(__)  ↙ 8-9 km/h     │    /(___(__)  ← 8-12 km/h    │   ― (   ) ―   ← 10-21 km/h   │
│      `-’      10 km          │     ⚡‘‘⚡‘‘  9 km           │     ⚡‘‘⚡‘‘  10 km          │      `-’      10 km              │
│     /   \     0.0 mm | 0%    │     ‘ ‘ ‘ ‘   0.0 mm | 76%   │     ‘ ‘ ‘ ‘   0.8 mm | 100%  │     /   \     0.0 mm | 0%    │
└──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
                                                       ┌─────────────┐                                                       
┌──────────────────────────────┬───────────────────────┤  Fri 24 Jan ├───────────────────────┬──────────────────────────────┐
│            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
│     \   /     Sunny          │     \   /     Sunny          │     \   /     Sunny          │     \   /     Clear          │
│      .-.      22 °C          │      .-.      &#43;26(27) °C     │      .-.      &#43;28(30) °C     │      .-.      &#43;22(25) °C     │
│   ― (   ) ―   ← 7-10 km/h    │   ― (   ) ―   ↙ 9-11 km/h    │   ― (   ) ―   ↙ 9-11 km/h    │   ― (   ) ―   ← 7-14 km/h    │
│      `-’      10 km          │      `-’      10 km          │      `-’      10 km          │      `-’      10 km          │
│     /   \     0.0 mm | 0%    │     /   \     0.0 mm | 0%    │     /   \     0.0 mm | 0%    │     /   \     0.0 mm | 0%    │
└──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
```

- Você pode especificar:
  - o nome da cidade e quando existir espaços basta usar `&#43;` e se quiser que o output tenha apenas o dia corrente, use o `1` precedido pelo sinal de interrogação `?` que é o delimitador que indica que qualquer coisa logo após, será uma opção de formatação de output... :
    - `curl wttr.in/Belo&#43;Horizonte`
      ```bash
      $ curl &#39;wttr.in/Belo&#43;Horizonte?1&#39;
      Weather report: Belo&#43;Horizonte
      
         _`/&#34;&#34;.-.     Thunderstorm, shower in vicinity
          ,\_(   ).   &#43;23(25) °C     
           /(___(__)  ↑ 9 km/h       
            ⚡‘‘⚡‘‘  10 km          
            ‘ ‘ ‘ ‘   14.9 mm        
                                                             ┌─────────────┐                                                       
      ┌──────────────────────────────┬───────────────────────┤  Wed 22 Jan ├───────────────────────┬──────────────────────────────┐
      │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
      ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
      │     \   /     Sunny          │  _`/&#34;&#34;.-.     Thundery outbr…│  _`/&#34;&#34;.-.     Thundery outbr…│    \  /       Partly Cloudy  │
      │      .-.      &#43;29(31) °C     │   ,\_(   ).   &#43;32(34) °C     │   ,\_(   ).   &#43;23(18) °C     │  _ /&#34;&#34;.-.     19 °C          │
      │   ― (   ) ―   ↘ 1 km/h       │    /(___(__)  → 7-8 km/h     │    /(___(__)  ↑ 9-20 km/h    │    \_(   ).   ↖ 6-13 km/h    │
      │      `-’      10 km          │     ⚡‘‘⚡‘‘  9 km           │     ⚡‘‘⚡‘‘  10 km          │    /(___(__)  10 km          │
      │     /   \     0.0 mm | 0%    │     ‘ ‘ ‘ ‘   0.0 mm | 0%    │     ‘ ‘ ‘ ‘   14.9 mm | 100% │               0.0 mm | 0%    │
      └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
      Location: BH, Microrregião Belo Horizonte, Região Metropolitana de Belo Horizonte, Mesorregião Metropolitana de Belo Horizonte, MG, Região Sudeste, Brasil [-19.9227318,-43.9450948]
      ```   

  - código de aeroportos:
    - `curl wttr.in/bhz`
      ```bash
      $ curl &#39;wttr.in/bhz?1&#39;
      Weather report: bhz
      
             .-.      Light rain
            (   ).    &#43;3(-2) °C      
           (___(__)   ↖ 21 km/h      
            ‘ ‘ ‘ ‘   10 km          
           ‘ ‘ ‘ ‘    0.0 mm         
                                                             ┌─────────────┐                                                       
      ┌──────────────────────────────┬───────────────────────┤  Thu 23 Jan ├───────────────────────┬──────────────────────────────┐
      │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
      ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
      │      .-.      Light rain     │  _`/&#34;&#34;.-.     Patchy rain ne…│    \  /       Partly Cloudy  │    \  /       Partly Cloudy  │
      │     (   ).    &#43;4(-1) °C      │   ,\_(   ).   &#43;3(-2) °C      │  _ /&#34;&#34;.-.     &#43;3(-1) °C      │  _ /&#34;&#34;.-.     &#43;2(-3) °C      │
      │    (___(__)   ↑ 28-42 km/h   │    /(___(__)  ↗ 27-40 km/h   │    \_(   ).   ↗ 24-41 km/h   │    \_(   ).   ↑ 23-38 km/h   │
      │     ‘ ‘ ‘ ‘   9 km           │      ‘ ‘ ‘ ‘  10 km          │    /(___(__)  10 km          │    /(___(__)  10 km          │
      │    ‘ ‘ ‘ ‘    1.1 mm | 100%  │     ‘ ‘ ‘ ‘   0.0 mm | 65%   │               0.0 mm | 0%    │               0.0 mm | 0%    │
      └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
      Location: BHZ, Stadtmitte, Ortsbeirat 14 : Stadtmitte, Rostock, Mecklenburg-Vorpommern, 18055, Deutschland [54.08250825,12.1426316547453]
      ```
  - o nome de atrações turísticas precedidas por til `~`:
    - `curl wttr.in/~Cristo&#43;Redentor`
      ```bash
      $ curl wttr.in/~Cristo&#43;Redentor?1
      Weather report: Cristo&#43;Redentor
      
            \   /     Clear
             .-.      &#43;27(30) °C     
          ― (   ) ―   ← 5 km/h       
             `-’      10 km          
            /   \     0.0 mm         
                                                             ┌─────────────┐                                                       
      ┌──────────────────────────────┬───────────────────────┤  Wed 22 Jan ├───────────────────────┬──────────────────────────────┐
      │            Morning           │             Noon      └──────┬──────┘     Evening           │             Night            │
      ├──────────────────────────────┼──────────────────────────────┼──────────────────────────────┼──────────────────────────────┤
      │    \  /       Partly Cloudy  │     \   /     Sunny          │  _`/&#34;&#34;.-.     Patchy rain ne…│     \   /     Clear          │
      │  _ /&#34;&#34;.-.     &#43;30(34) °C     │      .-.      &#43;32(36) °C     │   ,\_(   ).   &#43;28(32) °C     │      .-.      &#43;27(30) °C     │
      │    \_(   ).   ↓ 1 km/h       │   ― (   ) ―   ↖ 12-15 km/h   │    /(___(__)  ← 9-13 km/h    │   ― (   ) ―   ← 5-8 km/h     │
      │    /(___(__)  10 km          │      `-’      10 km          │      ‘ ‘ ‘ ‘  10 km          │      `-’      10 km          │
      │               0.0 mm | 0%    │     /   \     0.0 mm | 0%    │     ‘ ‘ ‘ ‘   0.0 mm | 64%   │     /   \     0.0 mm | 0%    │
      └──────────────────────────────┴──────────────────────────────┴──────────────────────────────┴──────────────────────────────┘
      Location: Cristo Redentor, Estrada do Corcovado, Santa Teresa, Zona Central do Rio de Janeiro, Rio de Janeiro, Microrregião Rio de Janeiro, Região Metropolitana do Rio de Janeiro, RJ, Região Sudeste, Brasil [-22.9519069,-43.2108581]
      ```
  - o nome DNS de domínio ou IP precedidos por arroba `@`:
    - `curl wttr.in/@tcpip.net.br` 
    - `curl wttr.in/@8.8.8.8`
  - a unidade meteorológica! Por default USCS é usado para consultas originadas dos USA e o métrico SI para o restante do mundo. Sendo...
    - a interrogação `?` é o delimitador que indica que qualquer coisa logo após, será uma opção de formatação de output... 
      - `u` para USCS: 
        - `curl &#39;wttr.in/Sao&#43;Paulo?u&#39;`
      - `m` para SI que é o default fora dos USA: 
        - `curl &#39;wttr.in/Belo&#43;Horizonte?m&#39;`  
      - `M` para SI porém com a amostragem da velocidade do vento em m/s: 
        - `curl &#39;wttr.in/Belo&#43;Horizonte?M&#39;`
      - por padrão, o output sempre será de 3 dias! Caso queira menos dias use qualquer opcção acima seguido de: `1` para apenas o dia corrente, ou `2` para dois dias. 
  - o idioma das legendas 
    - `curl &#39;wttr.in/Belo&#43;Horizonte?lang=pt&#39;`
  - `T` para forçar a saída em texto plano:
    - `curl wttr.in/Belo&#43;Horizonte?T`
  - `d` para restringir a saída a glifos disponíveis em fontes padrão do console:
    - `curl wttr.in/Belo&#43;Horizonte?T?d`  
  - a combinação de várias opções separando-as por `&amp;`:
    - aqui estou definindo a velocidade do vento em m/s e o idioma `pt`: 
      - `curl &#39;wttr.in/Belo&#43;Horizonte?M&amp;lang=pt&#39;`
  - saída de apenas uma linha! E que também é útil para integrar na barra de status de programas:
    - os formatos pré-configurados disponíveis são: 1, 2, 3, 4
      - 1: O tempo atual:
      ```bash
      $ curl wttr.in/Belo&#43;Horizonte?format=1
      ⛈   &#43;24°C
      ```
      - 2: o tempo atual com mais detalhes:
      ```bash
      $ curl &#39;wttr.in/Belo&#43;Horizonte?format=2&#39;
      ⛈   🌡️&#43;24°C 🌬️↖8km/h
      ```
      - 3: o nome do local e o clima: 
      ```bash
      $ curl &#39;wttr.in/Belo&#43;Horizonte?format=3&#39;
      Belo&#43;Horizonte: ⛈   &#43;24°C
      ```
      - 4: o nome do local, tempo atual e velocidade dos ventos:
      ```bash
      $ curl &#39;wttr.in/Belo&#43;Horizonte?format=4&#39;
      Belo&#43;Horizonte: ⛈   🌡️&#43;24°C 🌬️↖8km/h
      ```
    - Essea são pré-formatados! Mas se preferir, pode usar essas opções para criar o seu:
      ```text
      c    Weather condition,
      C    Weather condition textual name,
      x    Weather condition, plain-text symbol,
      h    Humidity,
      t    Temperature (Actual),
      f    Temperature (Feels Like),
      w    Wind,
      l    Location,
      m    Moon phase 🌑🌒🌓🌔🌕🌖🌗🌘,
      M    Moon day,
      p    Precipitation (mm/3 hours),
      P    Pressure (hPa),
      u    UV index (1-12),
  
      D    Dawn*,
      S    Sunrise*,
      z    Zenith*,
      s    Sunset*,
      d    Dusk*,
      T    Current time*,
      Z    Local timezone.
  
      (*times are shown in the local timezone)
      ``` 
      - Segue um exemplo de combinações
      ```bash
      $ curl &#39;wttr.in/Belo&#43;Horizonte?format=&#34;%l:&#43;%t&#43;%m&#43;%p&#43;%P&#43;%w&#43;%h&#43;%c&#43;%C&#43;%T\n&#39;
      &#34;Belo&#43;Horizonte: &#43;24°C 🌗 0.7mm 1017hPa ↖8km/h 78% ⛈   Shower in vicinity, thunderstorm in vicinity 20:27:41-0300
      ```
## Só isso ?
Não! RTFM [Wttr](https://github.com/chubin/wttr.in) para muito mais opções!

- Vou também pode acessar o --help via `curl`
  ```bash
  curl wttr.in/:help
  ```

     
### Segue uma sugestão de uso:

- Aqui estamos criando o arquivo `meuip_e_clima.sh` no diretório `~/.bashrc.d` para que sempre que abrir um novo terminal:
  - seja informado o seu IP de WAN atual.
  - o clima do dia e hora corrente.
  - e a criação de um alias na seção atual de nome `clima` que irá imprimir a saída padrão para a geolocalização do seu IP atual.

```bash
cat &lt;&lt;Uai&gt; ~/.bashrc.d/meuip_e_clima.sh

echo &#34;
     .... no! ...                  ... mno! ...
   ..... mno!! ...................... mnnoo! ...
 ..... mmno! ......................... mnnoo!! .
.... mnoonnoo!   mmmmmmmmmmpppoii!   mnno!!!! .
 ... !o! nno! mmmmmmmmmmmmmpppoooii!! no! ....
    ...... ! mmmmmmmmmmmmmppppooooiii! ! ...
   ........ mmmmmmmmmmmmpppppooooooii!! .....
   ........ mmmmmooooooppppppppoooomii! ...  
    ....... mmmmm..    oppmmp    .,omi! ....
     ...... mmmm::   o.,opmp,.o   ::i!! ...
         .... nnm:::.,,oopm!p,.::::!! ....
          .. mmnnnnnoooopmo!!iippo!!o! .....
         ... mmmmmnnnnoo:!!:!!ippppoo! ....
           .. mmmmmnnoommnniiipppoo!! ......
          ...... mmmonnmmnnniiioo!..........
       ....... mn mommmnnniiiiio! oo ..........
    ......... mno! iiiiiiiiiii oooo ...........
  ...... nnn.mno! . o!!!!!!!!!o . oono no! ........
   .... mnnnnno! ...ooooooooooo .  mmnnon!........
   ...... mnnnno! .. ppppppppp .. mmnon!........
      ...... oo! ................. on! .......
         ................................
&#34;

echo &#34;&gt;&gt;&gt; Get WAN data &#34;

# usando endpoint =  https://github.com/Fire-man-x/Public-IP-address---Gnome-shell-extension/blob/master/src/extension.js
curl -s https://ipv4.lookup.test-ipv6.com | jq &#39;{type,ip,asn,asn_name,country}&#39;

echo &#34;- - -&#34;

# Clima obtido pela geolocalização do seu IP wan
echo &#34;&gt;&gt;&gt; Get weather&#34; 
curl &#39;wttr.in?format=&#34;%l:&#43;%t&#43;%m&#43;%p&#43;%P&#43;%w&#43;%h&#43;%c&#43;%C&#43;%T\n&#39;

# criando alias &#34;clima&#34; 
alias clima=&#34;curl wttr.in&#34; 
                                                  
echo &#34;&gt;&gt;&gt; May the force be with you ${USER} ...o/&#34;
Uai
``` 





---

> Author: [Faioli a.k.a 0xttfx](https://github.com/0xttfx)  
> URL: http://0xttfx.tcpip.net.br/posts/rtfmaleatoriedades-climanoterminal/  

