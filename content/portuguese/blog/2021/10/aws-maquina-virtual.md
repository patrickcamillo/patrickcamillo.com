+++
title = "Como criar uma máquina virtual na AWS - O jeito fácil e rápido"
date = "2021-10-11T13:00:00-03:00"
draft = false
toc = true
comments = false
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech","aws","ptbr"]
+++

Olá! Neste artigo eu vou te ensinar a criar uma máquina virtual Linux na AWS. Eu disse no título que é o jeito fácil, pois existem formas de automatizar isso com outras ferramentas. Mas se você quer simplesmente começar logo, este artigo vai direto ao ponto.

Vamos criar uma máquina com a distribuição CentOS do Linux, em sua versão 7. Da forma como vamos fazer, ela ficará elegível para o [nível gratuito da AWS](https://aws.amazon.com/pt/free/) caso isso esteja disponível na sua conta.

## Requisitos

- Uma conta na AWS

Se ainda não tem uma conta, você pode acessar diretamente a página do [console](https://console.aws.amazon.com/), onde terá a opção de se cadastrar.

![Página de login do Console AWS](/images/aws-maquina-virtual/img5.png)

## Etapas

### 1. Entrando no serviço do EC2

Ao fazer login, você verá diversas sugestões para criação rápida de soluções. Vamos usar a **EC2**, que significa Elastic Compute Cloud. É o serviço de computação em nuvem da Amazon, que conta com uma configuração ao mesmo tempo simplificada e redimensioável, facilitando upgrades quando você precisar de mais capacidade.

Você pode digitar `EC2` na barra de pesquisa acima e clicar no primeiro resultado.

![Página inicial do Console AWS](/images/aws-maquina-virtual/img20.png)

Então verá a seguinte página:

![Console AWS, página inicial do serviço EC2](/images/aws-maquina-virtual/img6.png)

Clique na opção `Executar instâncias` no canto superior direito para começar.

### 2. Criando a máquina virtual

Na **etapa 1**, recomendo selecionar a opção `Somente nível gratuito` no menu lateral. Assim, especialmente se estiver fazendo testes, você garante que está selecionando a opção que não vai gerar custos inesperados. Neste guia vamos usar a distribuição CentOS, que você pode digitar na barra de pesquisa. Então pode selecionar a opção `AWS Marketplace` no menu lateral, assim:

![Criação de uma instância EC2 na AWS, passo 1](/images/aws-maquina-virtual/img7.png)

Vamos `selecionar` a primeira opção (CentOS 7 (x86_64) - with Updates HVM). Observe a informação `qualificado para o nível gratuito` ao lado.

A **etapa 2** é a seleção de tipo da instância, o que diz respeito às especificações de hardware que ela terá. Nesta etapa, você também verá a informação `qualificado para o nível gratuito` na instância `t2.micro`, que é a que vamos selecionar.

![Criação de uma instância EC2 na AWS, passo 2](/images/aws-maquina-virtual/img9.png)

A **etapa 3** pode ser avançada sem alterações. Você poderia selecionar a quantidade de instâncias que deseja executar (só precisamos de uma, o que é a opção marcada por padrão) e outras opções de rede. Tenha em mente que o nível gratuito oferece 750 horas mensais de instâncias EC2 sendo executadas. Se você executar **uma** instância durante o mês todo, 24 horas por dia, essa quota não será atingida. Porém se executar duas instâncias ou mais, somente a primeira será de nível gratuito e as demais serão cobradas. Cuidado nesse ponto.

Na **etapa 4** você pode configurar o armazenamento. Repare na mensagem que diz que usuários do nível gratuito podem obter até 30GB de armazenamento SSD. É o que vamos selecionar, como mostra a imagem:

![Criação de uma instância EC2 na AWS, passo 4](/images/aws-maquina-virtual/img10.png)

A **etapa 5** também pode ser avançada sem alterações. Nesta etapa é possível marcar a instância com identificadores para diversas finalidades. Particularmente não temos utilidade para isto agora.

A **etapa 6** serve para configurar algumas regras de segurança e acesso à máquina. Vamos deixar a opção `Criar um grupo de segurança novo` marcada, mantendo as opções padrão. **Opcionalmente** você pode clicar no botão `Adicionar regra` para criar duas novas, como na imagem abaixo:

![Criação de uma instância EC2 na AWS, passo 6](/images/aws-maquina-virtual/img11.png)

Neste exemplo, essas regras criadas servem para que um servidor web seja acessível externamente (um servidor web usa as portas 80 e 443 das regras que criamos). Se você já sabe de quais portas vai precisar, pode criar regras de acordo. Caso contrário, siga sem fazer isso.

Note que a regra de acesso indica que tudo e todos poderão acessar essa porta. A permissão `0.0.0.0/0` é totalmente aberta e deve ser evitada em um ambiente de produção, se possível. Você pode preencher com o seu IP atual para deixar as coisas mais seguras. Clique no botão `Verificar e ativar` para prosseguir.

Na tela da **etapa 7** a seguir, você pode revisar todas as opções selecionadas até aqui. Clique em `Executar`.

![Criação de uma instância EC2 na AWS, confirmação das opções selecionadas](/images/aws-maquina-virtual/img12.png)

### 3. Configurando o acesso à máquina

Você verá uma janela informando sobre a necessidade de usar um par de chaves para acessar a máquina. Recomendo selecionar a opção `criar um novo par de chaves`, escolher um nome e `fazer download`. Guarde bem o arquivo baixado. É com ele que vamos acessar a máquina e não é possível baixá-lo novamente no futuro. Por fim, clique em `Executar instâncias`.

![Criação de uma instância EC2 na AWS, criação de par de chaves](/images/aws-maquina-virtual/img13.png)

Depois de concluída a criação da instância, podemos voltar à página da EC2 no Console e ver a máquina nova na lista. Clicando nela, vemos as informações da imagem abaixo. Anote o IP da sua (na região em destaque).

![Console AWS, instância EC2 criada](/images/aws-maquina-virtual/img14.png)

## Acessando a máquina virtual

Para usar o arquivo de chave que baixamos no passo anterior, vou passá-lo para minha instalação do WSL no Windows. Se você está usando o Linux, não precisa fazer nada demais. Se estiver com o Windows e não tiver o WSL, você ainda pode usar o PuTTY, mas vai precisar configurar a chave nele com alguns passos a mais. Leia [este artigo](https://www.dialhost.com.br/ajuda/gravando-a-chave-publica-no-putty/) se precisar.

Coloquei o arquivo `.pem` no meu diretório `~/.ssh` do WSL e mudei as permissões para `600`, assim:

```
$ cp /mnt/d/Downloads/aws-proxy-reverso.pem .ssh/
$ cd .ssh/
$ chmod 600 aws-proxy-reverso.pem
$ ls -l
total 16
-rw------- 1 patrick patrick 1704 Oct  7 11:01 aws-proxy-reverso.pem
-rw------- 1 patrick patrick 1831 Aug 24 14:57 id_rsa
-rw-r--r-- 1 patrick patrick  405 Aug 24 14:57 id_rsa.pub
-rw-r--r-- 1 patrick patrick 3548 Oct  6 10:41 known_hosts
```

Então usamos o seguinte comando para acessar:
`ssh -i ~/.ssh/aws-proxy-reverso.pem centos@<IP_DA_MÁQUINA>`
Substitua `<IP_DA_MÁQUINA>` pelo IP que você anotou no fim do passo anterior.

Seu cliente SSH irá perguntar se você confirma a autenticidade do host. Responda `yes`. Em seguida, você estará conectado!

![Terminal conectado à máquina via SSH](/images/aws-maquina-virtual/img19.png)

## Verificando o custo das máquinas

Você pode sempre verificar o valor das faturas na [página de faturamento](https://console.aws.amazon.com/billing/). Enquanto sua conta for elegível para uso do nível gratuito, você verá algo parecido com isso nesta página:

![Página de faturamento (parcial)](/images/aws-maquina-virtual/img21.png)

Essa tabela mostra o quanto dos serviços gratuitos você está usando. No caso da máquina virtual, o que aparece são as informações de tempo de uso (gratuito até 750 horas por mês) e armazenamento utilizado (até 30GB).

## Concluindo

Obrigado por ler até aqui! Para um exemplo prático de uso de uma máquina virtual na AWS, escrevi [este artigo sobre proxy reverso](/blog/2021/10/proxy-reverso/). Você também pode ver a [tag 'aws' no meu site](/tags/aws).