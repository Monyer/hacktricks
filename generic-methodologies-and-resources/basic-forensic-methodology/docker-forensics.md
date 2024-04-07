# Forense do Docker

<details>

<summary><strong>Aprenda hacking da AWS do zero ao herói com</strong> <a href="https://training.hacktricks.xyz/courses/arte"><strong>htARTE (HackTricks AWS Red Team Expert)</strong></a><strong>!</strong></summary>

Outras maneiras de apoiar o HackTricks:

* Se você deseja ver sua **empresa anunciada no HackTricks** ou **baixar o HackTricks em PDF** Verifique os [**PLANOS DE ASSINATURA**](https://github.com/sponsors/carlospolop)!
* Adquira o [**swag oficial PEASS & HackTricks**](https://peass.creator-spring.com)
* Descubra [**A Família PEASS**](https://opensea.io/collection/the-peass-family), nossa coleção exclusiva de [**NFTs**](https://opensea.io/collection/the-peass-family)
* **Junte-se ao** 💬 [**grupo Discord**](https://discord.gg/hRep4RUj7f) ou ao [**grupo telegram**](https://t.me/peass) ou **siga-nos** no **Twitter** 🐦 [**@hacktricks_live**](https://twitter.com/hacktricks_live)**.**
* **Compartilhe seus truques de hacking enviando PRs para os** [**HackTricks**](https://github.com/carlospolop/hacktricks) e [**HackTricks Cloud**](https://github.com/carlospolop/hacktricks-cloud) repositórios do github.

</details>

## Modificação do Contêiner

Há suspeitas de que algum contêiner do Docker foi comprometido:
```bash
docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
cc03e43a052a        lamp-wordpress      "./run.sh"          2 minutes ago       Up 2 minutes        80/tcp              wordpress
```
Você pode facilmente **encontrar as modificações feitas neste contêiner em relação à imagem** com:
```bash
docker diff wordpress
C /var
C /var/lib
C /var/lib/mysql
A /var/lib/mysql/ib_logfile0
A /var/lib/mysql/ib_logfile1
A /var/lib/mysql/ibdata1
A /var/lib/mysql/mysql
A /var/lib/mysql/mysql/time_zone_leap_second.MYI
A /var/lib/mysql/mysql/general_log.CSV
...
```
No comando anterior, **C** significa **Alterado** e **A,** **Adicionado**.\
Se você descobrir que algum arquivo interessante como `/etc/shadow` foi modificado, você pode baixá-lo do contêiner para verificar atividades maliciosas com:
```bash
docker cp wordpress:/etc/shadow.
```
Você também pode **compará-lo com o original** executando um novo contêiner e extraindo o arquivo dele:
```bash
docker run -d lamp-wordpress
docker cp b5d53e8b468e:/etc/shadow original_shadow #Get the file from the newly created container
diff original_shadow shadow
```
Se você encontrar que **algum arquivo suspeito foi adicionado** você pode acessar o contêiner e verificá-lo:
```bash
docker exec -it wordpress bash
```
## Modificações em imagens

Quando você recebe uma imagem docker exportada (provavelmente no formato `.tar`), você pode usar [**container-diff**](https://github.com/GoogleContainerTools/container-diff/releases) para **extrair um resumo das modificações**:
```bash
docker save <image> > image.tar #Export the image to a .tar file
container-diff analyze -t sizelayer image.tar
container-diff analyze -t history image.tar
container-diff analyze -t metadata image.tar
```
Em seguida, você pode **descompactar** a imagem e **acessar os blobs** para procurar por arquivos suspeitos que você pode ter encontrado no histórico de alterações:
```bash
tar -xf image.tar
```
### Análise Básica

Você pode obter **informações básicas** da imagem em execução:
```bash
docker inspect <image>
```
Você também pode obter um resumo **histórico de alterações** com:
```bash
docker history --no-trunc <image>
```
Você também pode gerar um **dockerfile a partir de uma imagem** com:
```bash
alias dfimage="docker run -v /var/run/docker.sock:/var/run/docker.sock --rm alpine/dfimage"
dfimage -sV=1.36 madhuakula/k8s-goat-hidden-in-layers>
```
### Mergulhar

Para encontrar arquivos adicionados/modificados em imagens docker, você também pode usar a [**dive**](https://github.com/wagoodman/dive) (baixe-a em [**releases**](https://github.com/wagoodman/dive/releases/tag/v0.10.0)) utilitário:
```bash
#First you need to load the image in your docker repo
sudo docker load < image.tar                                                                                                                                                                                                         1 ⨯
Loaded image: flask:latest

#And then open it with dive:
sudo dive flask:latest
```
Isso permite que você **navegue pelos diferentes blobs das imagens do Docker** e verifique quais arquivos foram modificados/adicionados. **Vermelho** significa adicionado e **amarelo** significa modificado. Use **tab** para mover para a outra visualização e **espaço** para colapsar/abrir pastas.

Com die você não poderá acessar o conteúdo das diferentes etapas da imagem. Para fazer isso, você precisará **descompactar cada camada e acessá-la**.\
Você pode descompactar todas as camadas de uma imagem a partir do diretório onde a imagem foi descompactada executando:
```bash
tar -xf image.tar
for d in `find * -maxdepth 0 -type d`; do cd $d; tar -xf ./layer.tar; cd ..; done
```
## Credenciais da memória

Observe que ao executar um contêiner docker dentro de um host **você pode ver os processos em execução no contêiner a partir do host** apenas executando `ps -ef`

Portanto (como root) você pode **despejar a memória dos processos** do host e procurar por **credenciais** assim [**como no exemplo a seguir**](../../linux-hardening/privilege-escalation/#process-memory).