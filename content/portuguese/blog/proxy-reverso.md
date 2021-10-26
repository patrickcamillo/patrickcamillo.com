+++
title = "Como montar um proxy reverso com o nginx"
date = "2021-10-11T19:20:00-03:00"
draft = true
toc = true
comments = true
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech", "aws", "pt"]
+++

Olá! Este é um guia de como usar uma máquina virtual na AWS como proxy reverso para outro servidor. Isso vai ser um pouco extenso, então recomendo usar o índice acima para se localizar melhor.

## Requisitos

- Um serviço hospedado em algum servidor
- Uma conta na AWS
- Um domínio (e acesso para alterar os registros DNS deste)
- (Opcional) Uma conta no [FreeDNS](https://freedns.afraid.org/)

## O que faz um proxy reverso?

![Diagrama de um proxy reverso](/images/proxy-reverso/img22.png)

Um servidor de proxy reverso é responsável por receber solicitações, como o `app2.domínio.com` do diagrama acima. Quando essa solicitação está na lista de coisas que ele reconhece, ele redireciona o tráfego para outro lugar. Neste exemplo, essa solicitação seria redirecionada para a porta `10012` do servidor `srv.domínio.com`.

Isso pode ser feito de outras formas também. Você pode ter um servidor executando contâiners Docker em portas diversas porém não expostas externamente. No mesmo servidor, você pode ter um proxy reverso que irá aceitar requisições em diferentes subdomínios, direcionando cada uma delas para cada contâiner.

Neste artigo, vamos fazer como no diagrama, usando dois servidores.

## Exemplos de uso

A primeira situação em que você pode querer usar esse proxy reverso é quando está montando um [servidor em casa](https://patrickcamillo.com/servidor-em-casa/). A maioria das provedoras de internet bloqueiam certas portas do roteador fornecido por eles para clientes domésticos. Mesmo que você consiga entrar na página de configuração e criar os hosts virtuais, as portas continuam fechadas. Isso aconteceu comigo quando estava lidando com a Vivo e causou bastante frustração. As portas que eu mesmo testei foram 80, 443 e 8080, sendo as duas primeiras as mais importantes quando você está tentando hospedar um serviço web. Nesse caso, é possível liberar portas não padrão de numeração mais alta (eu mesmo usava portas acima do 10000), configurar os serviços para estas portas e usar uma VM do [nível gratuito da AWS](https://aws.amazon.com/pt/free/) caso ele esteja disponível para você. O que o proxy reverso vai fazer é interceptar o tráfego externo e redirecioná-lo para o meu servidor.

A segunda situação em que podemos usar essa solução é em um ambiente corporativo, como neste exemplo pessoal que eu enfrentei. Trabalhei para uma empresa que fornecia um software normalmente hospedado na infraestrutura dos clientes. Porém a empresa estava crescendo e passou a oferecer soluções em sua própria infraestrutura em caráter experimental. Essas soluções eram servidas na forma de portais de acesso hospedados em servidores web internos. O cenário legado que herdei continha diversos serviços em portas diferentes, que os clientes acessavam com URLs diferentes, como no exemplo:

```
https://portal.exemplo.com:8080
https://cloud.exemplo.com:9009
https://demo.exemplo.com:8443
```

Sendo todos eles CNAMEs para o mesmo host. Isso gera dois problemas: deselegância para o cliente, pois particularmente me incomoda muito acessar um serviço hospedado assim e; bagunça e confusão quanto mais os serviços crescem.
Por ser um cliente corporativo, a empresa tinha total liberdade para abrir e expôr somente as portas padrão para servidores web e deixar que o firewall (no meu caso, pfSense) ou um servidor de proxy lide com esses redirecionamentos.

Uma última informação antes de prosseguirmos: Tenha em mente que, ao usar uma VM na AWS como proxy reverso, todo o seu tráfego passa por ela. Isso não é um problema quando você usar aplicações web simples, mas se você começar a gerar muito tráfego, pode ser cobrado um valor bem alto. Isso acaba tornando esse método caro demais se você for montar um servidor de arquivos em casa, por exemplo.

## Configuração inicial - Os serviços locais

Para fins de demonstração, vou servir duas aplicações: O [Kanboard](https://kanboard.org/) na porta 10011 e uma página simples que eu fiz em HTML na porta 10012. O `.conf` dessas aplicações se parece com isso:

```
<VirtualHost *:10012>
        ServerName pckcml.test
        DocumentRoot /var/www/t1_colors
</VirtualHost>
```

Assim, quando eu acesso `http://192.168.25.150:10012` no meu navegador, vejo a seguinte página:

![Navegador exibindo página web em servidor local](/images/proxy-reverso/img2.png)

Normalmente você chega nesse ponto seguindo as instruções de instalação da aplicação que deseja instalar. O Kanboard tem resultado semelhante - eu só o escolhi porque outras aplicações que uso já me fizeram instalar todos os módulos do PHP que eu precisava.

Depois que a aplicação está funcionando de forma local, é necessário abrir as portas no roteador da ISP (no meu caso, Vivo) e testar o acesso externo. A minha tabela de virtual servers no router da Vivo fica assim:

![Tabela de virtual servers no router da Vivo](/images/proxy-reverso/img3.png)

E estou usando o serviço do [FreeDNS](https://freedns.afraid.org/) para não precisar chamar meu servidor por IP, até porque o IP não é fixo. Com isso, o acesso por celular à URL `http://patrickcamillo-rp-home.my.to:10011` no 4G fica assim:

![Navegador exibindo página web pelo celular](/images/proxy-reverso/img4.jpg)

Agora sim podemos prosseguir para a criação da máquina virtual na AWS.

## Executando a instância EC2 na AWS

Caso vá seguir pelo caminho da VM na AWS, você pode usar este guia para criar uma VM lá. Lembre-se de criar as regras de segurança liberando as portas 80 e 443, na etapa 6 deste guia linkado.

Se você for usar um segundo servidor que você já tem, pode prosseguir tendo em mente que daqui pra frente vou supor que você esteja usando a VM na AWS.

## Criando apontamento no DNS para a máquina virtual

Da mesma forma que criei um subdomínio para meu servidor em casa no FreeDNS alguns passos atrás, você pode fazer o mesmo para a VM no AWS. Basta criar um registro para cada serviço desejado, todos apontando para o IP do servidor proxy. Como eu tenho o meu domínio, vou criar hosts A para os serviços, como indicado na imagem abaixo:

![Route53, entrada no DNS criada](/images/proxy-reverso/img15.png)

## Instalando o nginx e configurando o serviço de proxy reverso

Se você usou a mesma imagem de máquina virtual que eu, você precisa instalar o `epel-release` e então instalar o `nginx`:
```
[centos@ip-172-31-40-85 ~]$ sudo yum install epel-release && sudo yum install nginx
```
Sendo que o primeiro serve para instalar o repositório `Extra Packages for Enterprise Linux`, pois os repositórios base do CentOS não contém o pacote do nginx.

Em seguida, vamos habilitar o serviço do nginx e configurá-lo para iniciar com o sistema:
```
[centos@ip-172-31-40-85 ~]$ sudo systemctl enable nginx
Created symlink from /etc/systemd/system/multi-user.target.wants/nginx.service to /usr/lib/systemd/system/nginx.service.
[centos@ip-172-31-40-85 ~]$ sudo systemctl start nginx
[centos@ip-172-31-40-85 ~]$ sudo systemctl status nginx
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

Oct 07 15:07:34 ip-172-31-40-85.us-east-2.compute.internal systemd[1]: Starting The nginx HTTP and reverse proxy se.....
Oct 07 15:07:34 ip-172-31-40-85.us-east-2.compute.internal nginx[9255]: nginx: the configuration file /etc/nginx/ng...ok
Oct 07 15:07:34 ip-172-31-40-85.us-east-2.compute.internal nginx[9255]: nginx: configuration file /etc/nginx/nginx....ul
Oct 07 15:07:34 ip-172-31-40-85.us-east-2.compute.internal systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
```

Para criarmos os arquivos de configuração, você precisa criar os diretórios `/etc/nginx/sites-available` e `/etc/nginx/sites-enabled`, e então editar o arquivo `/etc/nginx/nginx.conf`: Este arquivo tem um bloco identificado por `http`, dentro do qual você precisa inserir a linha:
```
include /etc/nginx/sites-enabled/*;
```

A forma como a configuração vai funcionar é a seguinte: Criaremos arquivos dentro de `sites-available` para configurar o servidor, e então criaremos links simbólicos desse diretório para o diretório `sites-enabled`. Esse diretório agora é verificado pelo ngnix graças à linha que acabamos de inserir no arquivo de configuração dele.

Agora vamos criar os arquivos `/etc/nginx/sites-available/rp-kanboard.conf` e `/etc/nginx/sites-available/rp-colors.conf` para fazer a nossa configuração. O conteúdo destes arquivos inicialmente será, respectivamente:
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

E então execute os seguintes comandos para criar um link simbólico desses arquivos:
```
[centos@ip-172-31-40-85 ~]$ sudo ln -s /etc/nginx/sites-available/kanboard.conf /etc/nginx/sites-enabled/kanboard.conf
[centos@ip-172-31-40-85 ~]$ sudo ln -s /etc/nginx/sites-available/colors.conf /etc/nginx/sites-enabled/colors.conf
```

Ao final, podemos executar o comando abaixo para testar as configurações e reiniciar o serviço do nginx para que as mesmas sejam aplicadas:
```
[centos@ip-172-31-40-85 ~]$ sudo nginx -t && sudo systemctl restart nginx
```

## Editando configuração de segurança do SELinux

Se você seguiu perfeitamente os passos até aqui, vai notar que ainda não é possível usar a URL do servidor de proxy reverso. Caso faça isso, vai se deparar com a tela a seguir:

![Route53, entrada no DNS criada](/images/proxy-reverso/img16.png)

Pelo menos foi o que aconteceu comigo! Caso também tenha acontecido com você, veja a linha de investigação que eu segui abaixo. A solução está no final desta seção.

Entrei com o usuário `root` para ler mais facilmente os logs do nginx, localizados em `/var/log/nginx/`. O primeiro arquivo que investiguei foi o `access.log`. Usando o comando `tail -f -n 0 /var/log/nginx/access.log` e depois disso atualizando a página, vi o seguinte resultado:
```
<MEU_IP> - - [07/Oct/2021:20:42:02 +0000] "GET / HTTP/1.1" 502 157 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:93.0) Gecko/20100101 Firefox/93.0" "-"
```
Tudo certo por aqui - O servidor sabe que eu estou requisitando a página.

Já no arquivo `error.log`, vi o seguinte:
```
2021/10/07 20:43:12 [crit] 9685#9685: *77 connect() to <IP_DO_MEU_SERVIDOR>:10012 failed (13: Permission denied) while connecting to upstream, client: <MEU_IP>, server: rp-colors.patrickcamillo.com, request: "GET / HTTP/1.1", upstream: "http://<IP_DO_MEU_SERVIDOR>:10012/", host: "rp-colors.patrickcamillo.com"
```

Temos algo! Pesquisei um pouco e encontrei [este post no blog do nginx](https://www.nginx.com/blog/using-nginx-plus-with-selinux/). Ele nos diz que o problema envolve o SELinux. Não tenho vergonha de admitir que na data de escrita deste artigo, não tenho muito conhecimento de como funciona o SELinux. Mas o artigo no post já nos dá uma boa introdução. Vamos analisar.

>O SELinux vem habilitado por padrão em servidores RHEL e CentOS modernos. Cada objeto do sistema operacional (processo, descritor de arquivo, arquivo etc) é rotulado com um contexto do SELinux que define as permissões e operações que um objeto pode realizar. Do RHEL 6.6/CentOS 6.6 em diante, NGINX é rotulado com o contexto `httpd_t`.

Legal. Com isso, aprendemos por alto como o SELinux funciona e como exatamente ele está atuando sobre o nginx. O artigo continua a explicar que o SELinux concede diversas permissões para o ngnix funcionar, mas a operação de proxy para localizações upstream na rede (no sentido de que o nginx não pode, como cliente, comunicar-se com um servidor).

O artigo mostra como é possível desabilitar a atuação completa do SELinux sobre o nginx de forma temporária ou permanente. Mas isso não é muito legal, pois pode abrir brechas de segurança desnecessárias e perigosas. Em nenhum contexto é interessante simplesmente desabilitar a segurança completamente (lembre-se das permissões de rede da instância EC2 mais acima neste artigo).

Em vez de desabilitar completamente, podemos investigar precisamente qual o parâmetro do SELinux que está impactando. Na minha pesquisa, descobri que ele pode ser diferente mesmo em cenários semelhantes. Então você pode querer investigar o seu problema específico, como vou fazer a seguir.

Execute o comando `tail -f -n 0 /var/log/audit/audit.log` e tente acessar novamente a página. Uma das mensagens que aparece é a seguinte:
```
type=AVC msg=audit(1633647336.240:1355): avc:  denied  { name_connect } for  pid=9685 comm="nginx" dest=10012 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
```

Podemos encontrar a explicação dessa mensagem dada pelo próprio sistema, usando o pacote `audit2why` e passando a mensagem de erro, da seguinte maneira: `grep 1633647336.240:1355 /var/log/audit/audit.log | audit2why`. A saída é a seguinte:
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

A solução que podemos aplicar é o primiero comando sugerido pela ferramenta: `setsebool -P httpd_can_network_connect 1`, onde vamos ativar essa variável para "permitir que módulos e scripts HTTP iniciem conexões com uma rede ou porta remota", definição retirada [desta documentação](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-managing_confined_services-the_apache_http_server-booleans).

## Concluindo

Finalmente, podemos então acessar a página `http://rp-colors.patrickcamillo.com/` e ver o seguinte resultado:

![Resultado final: Proxy reverso funcionando para a página "colors"](/images/proxy-reverso/img17.png)

O mesmo para a página `http://rp-kanboard.patrickcamillo.com/`:

![Resultado final: Proxy reverso funcionando para a página "kanboard"](/images/proxy-reverso/img18.png)

---

### Referências

[Getting started with self hosting - episode 4](https://mickael.kerjean.me/2018/03/14/getting-started-with-selfhosting-episode-4/)
[NGINX Reverse Proxy Documentation](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
[Beginner’s Guide to NGINX Configuration Files](https://medium.com/adrixus/beginners-guide-to-nginx-configuration-files-527fcd6d5efd)