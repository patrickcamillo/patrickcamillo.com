+++
title = "Um servidor em casa"
date = "2021-09-21T08:04:44-03:00"
draft = false
toc = true

#
# description is optional
#
# description = "An optional description for SEO. If not provided, an automatically created summary will be used."

tags = ["tech", "pt"]
+++

Nos últimos meses aprendi a configurar um servidor Linux do zero, sem interface gráfica, e estive usando aplicações hospedadas nele para facilitar algumas coisas na minha vida. O motivo para fazer assim - em vez de pagar um dos vários serviços que já existem - foi por pura vontade de aprender e fazer acontecer.

Este artigo não tem intenção de ser um tutorial passo-a-passo de como montar um servidor assim. Use-o como inspiração caso queira fazer em casa. E claro, se tiver alguma dúvida ou quiser mais detalhes sobre algo específico, não deixe de entrar em contato comigo!

## Introdução

Quando falo que tenho um servidor, as pessoas que sabem o que é um servidor se contorcem por dentro. Os servidores usados em empresas realmente são enormes, pesados, e barulhentos demais para se ter em um quarto. Imagina dormir com o som do cooler, o calor 24/7 e as luzes piscando! Porém qualquer computador pode ser um servidor para fins caseiros. Quanto menor, melhor! Eis o meu setup:

![Odroid HC1 ao lado de uma caneta Pilot](/images/servidor-em-casa/img1.jpg)

Estou usando um Odroid HC1, que tem uma placa BEM pequena e com apenas algumas conexões. Ele também tem o espaço para um HD de notebook ou SSD. Não vejo muito à venda aqui no Brasil, até porque ele foi [descontinuado](https://www.hardkernel.com/shop/odroid-hc1-home-cloud-one/), mas dei sorte de pagar apenas R$ 220,00 no Mercado Livre em meados de junho de 2020.

O sistema operacional é executado a partir de um cartão SD (que você precisa gravar em um computador antes) e, ao ligar o HC1 na tomada e na rede, ele vai receber um IP por DHCP. Você precisa então entrar no roteador para vê-lo dentre os leases e então conectar via SSH.

Super fácil até aqui quando você leva em conta que ele consome menos de 10 Watts em idle. No fundo, também existe o conceito de liberdade, no sentido de que meu servidor está no meu quarto e os meus serviços estão 100% sob meu controle. Pretendo escrever um artigo sobre minha relação com a privacidade digital no futuro, mas já adianto que isso é bem importante pra mim.

## Aplicações que eu uso

Desde o início eu já tinha uma ideia de quais aplicações gostaria de hospedar no servidor, e agora, pouco mais de um ano depois, cheguei no seguinte conjunto de soluções, todas elas gratuitas, livres e de código aberto:

### [Kimai](https://www.kimai.org/)

Essa é uma aplicação web que serve para controle de horas trabalhadas. Eu costumava sair para vários clientes ao longo do dia, então foi muito melhor controlar por aqui do que por uma planilha do Google Docs.

### [BookStack](https://www.bookstackapp.com/)

Outra aplicação web. Serve como uma wiki organizada hierarquicamente. Em vez de pensar em páginas com infinitas subpáginas, ela simplifica a organização com o seguinte conceito:

Prateleiras -> Livros -> Capítulos -> Páginas.

Acabei usando mais como journal mesmo, tendo em vista que não queria encher uma aplicação pessoal com coisas de trabalho. Não tive a oportunidade de implementar na empresa porque o pessoal usa soluções Microsoft, mas é uma boa ideia para documentar ambientes de trabalho. É possível adicionar diversos usuários no sistema, controlar algumas permissões, comentar em páginas e ver histórico de edições.

### [Firefly III](https://www.firefly-iii.org/)

Esse é um grande queridinho. Faz quatro anos que eu controlo minhas finanças pessoais. Comecei com planilha no Google Docs, que depois passou para um sistema pago chamado Mobills. Este é bom para um controle básico, mas criei aversão a ele quando percebi que a ferramenta de exportação de dados não exportava transferência entre contas (coisa que eu fazia bastante). Exportar somente entradas e saídas deixaria um grande buraco nos meus dados. Acabei criando um programa em JavaScript para exportar meus dados (pretendo finalizar e compartilhar no futuro) e migrei tudo para o Firefly.

Só precisei aprender alguns conceitos novos - como o fato de que cartões de crédito são interpretados dentro do sistema como contas de ativos que sempre ficam negativas, e você precisa fazer transferências mensais para representar as faturas sendo pagas. Pessoalmente acho muito importante fazer esse tipo de controle e não tenho nada a reclamar do Firefly. Ele tem boas funcionalidades de relatórios e, **principalmente**, permite exportar todas as transações, diferente do Mobills.

### [Radicale](https://radicale.org/3.0.html)

Essa é uma aplicação mais simples, que nem tem front-end na web. É um servidor CalDAV para armazenar contatos e calendários. Em vez de entregar e confiar no Google ou qualquer outro serviço por aí, deixo meus contatos sempre sob meu controle. Então uso uma aplicação como o DAVx5 no celular para sincronizar com o servidor periodicamente.

## Como isso funciona

Todas as aplicações dessa lista funcionam com uma [LAMP stack](https://www.ibm.com/cloud/learn/lamp-stack-explained) (Linux, Apache, MySQL e PHP), exceto o Radicale, que trabalha com Python. A documentação de todos os serviços é bem compreensiva e prática de seguir. Porém é nesse ponto que preciso adicionar um *disclaimer*. Com meu interesse recente na cultura DevOps, eu não faria esse mesmo setup se estivesse começando hoje. Todas essas quatro aplicações têm containers Docker disponíveis, sempre bem atualizados. Subir um serviço totalmente na mão assim tem seus perigos em um contexto enterprise - riscos totalmente evitáveis se você souber configurar e seguir as boas práticas usando containeres.

Porém é claro que para um servidor caseiro não tem problema nenhum subir dessa maneira. Na verdade, é até bom fazer assim em um primeiro contato, pois você realmente compreende como funciona uma aplicação web - como configurar um Apache host file, como o Apache vai abrir as portas, como criar e configurar um certificado SSL, como instalar módulos PHP para cumprir os pré-requisitos, como funciona um arquivo `.env`, como criar usuários no MySQL, como fazer backup disso tudo etc.

## Seguindo adiante

Nos próximos meses pretendo estudar Docker e AWS. Com isso, vai ser útil subir uma instância EC2 (especialmente se puder ser no tier gratuito) para aprender as funcionalidades de lá. Penso em dedicar uma instância para substituir o meu servidor caseiro por um motivo muito simples: não é possível abrir as portas 80 e 443 do roteador da Vivo. Com isso, todas as minhas aplicações ficavam com uma URL como a seguinte: `https://domínio.com:9011/kimai`. Não existe nenhum problema em usar assim, exceto o fato de que não é **nada elegante**. Uma das maneiras de resolver isso é usando um [proxy reverso](https://patrickcamillo.com/proxy-reverso), porém eu mesmo não uso essa solução porque estou contente me conectando à minha rede com uma VPN do Wireguard hospedada em um router que roda [OpenWRT](https://openwrt.org/). Yep, adoro essas coisas. Pretendo escrever um artigo sobre isso, porque enfreitei problemas para configurar o Wireguard e acho que isso vai ajudar algumas pessoas.

Não pretendo aposentar o HC1 - pelo contrário. Ele continua sendo extremamente semelhante ao Raspberry Pi, o que me permite executar diversos projetos existentes internet afora com uma boa compatibilidade. Vamos ver onde isso vai me levar.