+++
title = "Usando o NGINX como proxy reverso"
date = "2021-10-27T10:00:00-03:00"
draft = false
toc = true
comments = false
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech","aws","linux","ptbr"]
+++

Este é um guia de como usar o NGINX como proxy reverso. Isso vai ser um pouco extenso, então recomendo usar o índice acima caso queira avançar para uma etapa específica.

## Requisitos

**Mínimos**  
- Um servidor com Linux

**Adicionais**  
- Uma conta na AWS
- Um domínio
- Uma conta no [FreeDNS](https://freedns.afraid.org/)

## O que faz um proxy reverso?

![Diagrama de um proxy reverso](/images/proxy-reverso/img22.png)

Um servidor de proxy reverso é responsável por receber solicitações HTTP (como `app2.domínio.com` no diagrama acima). Quando essa solicitação está na lista de servidores que o proxy reverso reconhece, ele redireciona o tráfego para outro lugar. Neste exemplo, a solicitação `app2.domínio.com` seria redirecionada para a porta `10012` do servidor `srv.domínio.com`.

Um exemplo de aplicação prática é quando você tem um servidor com diversos containers Docker. Em outro servidor, você pode ter um proxy reverso que irá aceitar requisições em diferentes subdomínios, direcionando cada um deles para cada container correspondente no primeiro servidor.

Também é possível que o proxy reverso esteja no **mesmo** servidor da aplicação. Neste caso, as portas usadas pelas aplicações ficam restritas somente para acesso interno e o proxy reverso se encarrega de ficar na borda esperando conexões e redirecionando de acordo.

Neste artigo, vou dar exemplos dos dois setups: usando um ou dois servidores.

## Mais exemplos de uso real antes de começar

Você pode querer usar um proxy reverso quando está montando um [servidor em casa](/blog/2021/09/servidor-em-casa//). A maioria das provedoras de internet impede a abertura de certas portas do roteador fornecido para clientes domésticos. Mesmo que você consiga entrar na página de configuração e criar os hosts virtuais, as portas continuam fechadas. Isso aconteceu comigo quando estava lidando com a Vivo e causou bastante frustração. As portas que eu testei foram 80, 443 e 8080, as duas primeiras sendo as mais importantes quando você está tentando hospedar um serviço web. Neste caso, é possível liberar portas não padrão de numeração mais alta (como portas acima de 1024), servir as páginas por essas portas e usar uma VM do [nível gratuito da AWS](https://aws.amazon.com/pt/free/) como proxy reverso. O que o proxy reverso vai fazer é interceptar o tráfego externo e redirecioná-lo para o servidor em casa.

---

Outra situação que enfrentei foi em um ambiente corporativo. Trabalhei para uma empresa que fornecia soluções em sua própria infraestrutura para os clientes. Essas soluções eram hospedadas em servidores web, que os clientes acessavam com URLs parecidas com o exemplo abaixo:

```
https://portal.exemplo.com:8080
https://cloud.exemplo.com:9009
https://demo.exemplo.com:8443
```

Todos eram CNAMEs para o mesmo host. Isso gera dois problemas: a presença da porta na URL parece "gambiarra", pois particularmente me incomoda muito acessar um serviço hospedado assim e; isso também gera bagunça e confusão quanto mais os serviços crescem.
Por ser um cliente corporativo, a empresa tinha total liberdade para usar as portas 80 e 443 e deixar que um servidor de proxy reverso lidasse com esses redirecionamentos internamente.

## Setup com apenas um servidor

### Serviços locais para demonstração

Para fins de demonstração, vou servir duas aplicações baseadas em Python. Uma será um simples hello world em [Flask](https://flask.palletsprojects.com/en/2.0.x/quickstart/) e a outra será um sistema de comentários chamado [Isso](https://posativ.org/isso/) (o mesmo utilizado neste site). As portas utilizadas serão a `5000` e a `8011`, respectivamente. Veja o que acontece quando acesso as aplicações direto pelo IP e porta:

![Navegador exibindo páginas web em servidor local](/images/proxy-reverso/img23.png)

Tudo funciona, mas queremos evitar ter que digitar IP e porta para acessar as páginas. Tua situação particular vai ser diferente, mas suponho que seja o mesmo cenário: aplicações sendo servidas em portas diversas. Vamos para a parte do proxy reverso.

### Instalando o NGINX e configurando o proxy reverso

Estou usando o CentOS 7. Antes de instalar o NGINX, preciso instalar o pacote `epel-release` porque o NGINX não vem nas fontes padrão. Pesquise os detalhes específicos caso utilize outra distribuição. Então instale o pacote do NGINX:
```
$ sudo yum install nginx
```

Em seguida, vamos habilitar o serviço do NGINX e configurá-lo para iniciar com o sistema:
```
$ sudo systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.

$ sudo systemctl start nginx

$ sudo systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2021-10-07 15:07:34 UTC; 14s ago
  Process: 9257 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 9255 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 9254 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 9259 (nginx)
   CGroup: /system.slice/nginx.service
           ├─9259 nginx: master process /usr/sbin/nginx
           └─9261 nginx: worker process
```

---

De acordo com a configuração padrão do NGINX (localizada em `/etc/nginx/nginx.conf`), o serviço procura por arquivos de configuração com a extensão `.conf` no diretório `/etc/nginx/conf.d/`. É lá que vamos criar um arquivo para cada site.

Vamos criar os arquivos `/etc/nginx/conf.d/flask.conf` e `/etc/nginx/conf.d/isso.conf`. O conteúdo destes arquivos será, respectivamente:
```
server {
    listen 80;
    server_name flask.pckcml.test;
    location / {
        proxy_pass http://localhost:5000;
    }
}
```
```
server {
    listen 80;
    server_name isso.pckcml.test;
    location / {
        proxy_pass http://localhost:8011;
    }
}
```

Observe que o `server_name` de ambos termina em `pckcml.test`. Eu tenho um roteador que me permite criar registros DNS, então eu apontei ambos para o IP desse servidor. Assim:

![Registros DNS locais](/images/proxy-reverso/img24.png)

No seu caso, pode ser que você tenha um domínio externo. Use o DNS desse domínio e crie hosts A ou CNAMEs de acordo.

Toda vez que alterar os arquivos do NGINX, execute o comando abaixo para testar as configurações e reiniciar o serviço para aplicar o que foi alterado:
```
$ sudo nginx -t && sudo systemctl restart nginx
```

### Testando o acesso aos serviços com o novo endereço

Se tudo estiver correto, será possível acessar os endereços `flask.pckcml.test` e `isso.pckcml.test`, assim:

![Navegador exibindo páginas web em servidor local](/images/proxy-reverso/img25.png)

Este é o setup mais simples do NGINX, usando um único servidor tanto para as aplicações quanto para o proxy reverso. A situação com dois servidores é ligeiramente diferente.

## Setup com dois servidores

Nesta parte, vou hospedar algumas páginas web no meu servidor em casa. Depois disso, vou usar uma [máquina virtual na AWS](/blog/2021/10/aws-maquina-virtual/) para hospedar o proxy reverso. Também vou usar o DNS do meu domínio.

Uma observação antes de prosseguirmos: Tenha em mente que, ao usar uma VM na AWS como proxy reverso, todo o seu tráfego passa por ela. Isso não é um problema quando você usa aplicações web simples, mas se você começar a gerar muito tráfego, pode ser cobrado um valor bem alto. Isso acaba tornando esse método caro demais se você for montar um servidor de arquivos em casa, por exemplo, pois ele tende a gerar um volume de dados muito maior do que o nível gratuito permite. Cuidado para não gerar custos inesperados!

### Serviços locais para demonstração

Para fins de demonstração, vou servir duas aplicações: O [Kanboard](https://kanboard.org/) na porta `10011` e uma página simples que eu fiz em HTML na porta `10012`. Neste servidor de aplicação vou usar o Apache, apenas para variar um pouco. O `.conf` dessas aplicações se parece com isso:

```
<VirtualHost *:10012>
        DocumentRoot /var/www/t1_colors
</VirtualHost>
```

Assim, quando eu acesso `http://192.168.25.150:10012` no meu navegador, vejo a seguinte página:

![Navegador exibindo página web em servidor local](/images/proxy-reverso/img2.png)

### Configuração para acesso externo

Depois que a aplicação está funcionando de forma local, é necessário abrir as portas no roteador da ISP (no meu caso, Vivo) e testar o acesso externo. A minha tabela de servidores virtuais no roteador da Vivo fica assim:

![Tabela de servidores virtuais no roteador da Vivo](/images/proxy-reverso/img3.png)

Também estou usando o serviço do [FreeDNS](https://freedns.afraid.org/) para não precisar chamar meu servidor por IP. Criei uma entrada no FreeDNS com o endereço `patrickcamillo-rp-home.my.to` apontando para o meu IP externo. Então acessei a URL `http://patrickcamillo-rp-home.my.to:10011` pelo 4G no celular, para garantir que esteja funcionando externamente. Fica assim:

![Navegador exibindo página web pelo celular](/images/proxy-reverso/img4.jpg)

### Configuração do NGINX na máquina virtual da AWS

A instalação do NGINX na VM segue o mesmo método indicado [mais acima neste artigo](#instalando-o-nginx-e-configurando-o-proxy-reverso). Porém os arquivos `.conf` dos sites devem apontar para o meu servidor em casa, e não para o `localhost`. Mas o conceito é o mesmo:

```
server {
    listen 80;
    server_name rp-kanboard.patrickcamillo.com;
    location / {
        proxy_pass http://patrickcamillo-rp-home.my.to:10011;
    }
}
```
```
server {
    listen 80;
    server_name rp-colors.patrickcamillo.com;
    location / {
        proxy_pass http://patrickcamillo-rp-home.my.to:10012;
    }
}
```

Note que existem dois `server_name`: `rp-kanboard` e `rp-colors`. Eu criei ambos como hosts A no meu DNS, assim:

![Route53, entrada no DNS criada](/images/proxy-reverso/img15.png)

Antes de concluir e finalmente acessar as páginas, enfrentei um pequeno problema. Vamos analisar e resolver.

### Possível problema com o NGINX

Se você olhar novamente o diagrama que mostrei no começo do artigo, vai notar que o servidor de proxy (o que tem o NGINX) precisa encaminhar dados para um servidor externo (o que tem as aplicações em si). Porém você pode se deparar com esta tela ao tentar acessar o servidor proxy:

![Erro "502 Bad Gateway" no NGINX](/images/proxy-reverso/img16.png) 

Pelo menos foi o que aconteceu comigo! Caso também tenha acontecido com você, veja a linha de investigação que eu segui abaixo. A solução está no final desta seção.

---

Entrei com o usuário `root` para ler mais facilmente os logs do NGINX, localizados em `/var/log/nginx/`. O primeiro arquivo que investiguei foi o `access.log`. Executei o comando `tail -f -n 0 /var/log/nginx/access.log` e depois disso atualizei a página web. Vi o seguinte resultado no terminal:
```
<MEU_IP> - - [07/Oct/2021:20:42:02 +0000] "GET / HTTP/1.1" 502 157 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:93.0) Gecko/20100101 Firefox/93.0" "-"
```
Tudo certo por aqui. O servidor sabe que eu estou requisitando a página.

Já no arquivo `error.log`, vi o seguinte:
```
2021/10/07 20:43:12 [crit] 9685#9685: *77 connect() to <IP_DO_MEU_SERVIDOR>:10012 failed (13: Permission denied) while connecting to upstream, client: <MEU_IP>, server: rp-colors.patrickcamillo.com, request: "GET / HTTP/1.1", upstream: "http://<IP_DO_MEU_SERVIDOR>:10012/", host: "rp-colors.patrickcamillo.com"
```

Temos algo! Ver um erro é sempre melhor do que não ver nada. Pesquisei um pouco e encontrei [este post no blog do NGINX](https://www.nginx.com/blog/using-nginx-plus-with-selinux/). Ele nos diz que o problema envolve o **SELinux**. Não tenho vergonha de admitir que na data de escrita deste artigo, não tenho muito conhecimento de como funciona o SELinux. Mas o post já nos dá uma boa introdução:

>O SELinux vem habilitado por padrão em servidores modernos RHEL e CentOS. Cada objeto do sistema operacional (processo, descritor de arquivo, arquivo etc) é rotulado com um contexto do SELinux que define as permissões que o objeto tem e operações que ele pode realizar. Do RHEL 6.6/CentOS 6.6 em diante, NGINX é rotulado com o contexto `httpd_t`.

Legal. Com isso, aprendemos por alto como o SELinux funciona e como exatamente ele está atuando sobre o NGINX. O artigo continua a explicar que o SELinux concede diversas permissões para o NGINX funcionar, mas a operação de proxy para localizações upstream na rede não é permitida (no sentido de que o NGINX não pode, como cliente, comunicar-se com um servidor externo).

O artigo mostra como é possível desabilitar a atuação completa do SELinux sobre o NGINX de forma temporária ou permanente. Mas isso não é muito legal, pois pode abrir brechas de segurança desnecessárias e perigosas. Em nenhum contexto é interessante desabilitar a segurança completamente. Podemos investigar precisamente qual o parâmetro do SELinux que está impedindo a conexão e alterar *somente* ele.

Executei o comando `tail -f -n 0 /var/log/audit/audit.log` e tentei acessar novamente a página. Uma das mensagens que aparece no terminal é a seguinte:
```
type=AVC msg=audit(1633647336.240:1355): avc:  denied  { name_connect } for  pid=9685 comm="nginx" dest=10012 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Podemos pesquisar a explicação desta mensagem no próprio sistema, usando o pacote `audit2why`. Executei este comando: `grep 1633647336.240:1355 /var/log/audit/audit.log | audit2why`. A saída dele é a seguinte:
```
Was caused by:
One of the following booleans was set incorrectly.
Description:
Allow httpd to can network connect

Allow access by executing:
# setsebool -P httpd_can_network_connect 1
Description:
Allow nis to enabled

Allow access by executing:
# setsebool -P nis_enabled 1
```

O primeiro comando sugerido pela ferramenta serve para ativar uma variável `httpd_can_network_connect`. Ela "permite que módulos e scripts HTTP iniciem conexões com uma rede ou porta remota", definição que eu retirei [desta documentação](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans).

Então, basta executar o comando `setsebool -P httpd_can_network_connect 1` e podemos partir para a próxima etapa.

### Testando o acesso aos serviços com o novo endereço

Finalmente, pude acessar a página `http://rp-colors.patrickcamillo.com/` e ver o seguinte resultado:

![Resultado final: Proxy reverso funcionando para a página "colors"](/images/proxy-reverso/img17.png)

O mesmo para a página `http://rp-kanboard.patrickcamillo.com/`:

![Resultado final: Proxy reverso funcionando para a página "kanboard"](/images/proxy-reverso/img18.png)

## Conclusão

Este artigo ensinou como funciona um proxy reverso e como se configura um desses usando o NGINX. Aprendemos juntos também o funcionamento básico do SELinux. Ambas ferramentas possuem diversas diretrizes e configurações que valem muito a pena pesquisar mais a fundo e entender como funcionam.

Se você já tem algum conhecimento de servidores web, notou que eu não cheguei a usar HTTPS, mesmo em um serviço exposto externamente. Estou reservando isso para um artigo no qual vou me aprofundar mais em algumas configurações do NGINX, bem como a parte de gerar um certificado SSL/TLS gratuito com o Let's Encrypt. Aguarde novidades!

Em todo caso, muito obrigado por acompanhar até aqui. Como sempre, se tiver alguma dúvida não deixe de usar os comentários abaixo. Valeu!

---

## Referências

[Getting started with self hosting - episode 4](https://mickael.kerjean.me/2018/03/14/getting-started-with-selfhosting-episode-4/)  
[NGINX Reverse Proxy Documentation](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)  
[Using NGINX and NGINX Plus with SELinux](https://www.nginx.com/blog/using-nginx-plus-with-selinux/)  
[SELinux User's and Administrator's Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans)