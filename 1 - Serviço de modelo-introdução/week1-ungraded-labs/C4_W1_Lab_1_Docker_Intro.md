Adaptado de [Machine Learning in Production](https://www.deeplearning.ai/courses/machine-learning-in-production/) de [Andrew Ng](https://www.deeplearning.ai/)  ([Stanford University](http://online.stanford.edu/), [DeepLearning.AI](https://www.deeplearning.ai/))


# Introdução ao Docker e Instalação

Neste laboratório e em outros posteriores, você aprenderá diferentes técnicas que permitirão implantar modelos de aprendizado de máquina.

Há uma lacuna de conhecimento entre o treinamento bem-sucedido de um modelo em um ambiente local e a sua utilização efetiva..  

---------
## Docker
Utilizaremos muito o [Docker](https://www.docker.com/) por isso **será necessário que você execute o código em seu computador local**.

### Porquê Docker?

O Docker é uma ferramenta incrível que permite que você **transporte seu software junto com todas as suas dependências**. Isso é ótimo porque permite que você execute o software mesmo sem instalar os interpretadores ou compiladores necessários para que ele seja executado.

Vamos usar um exemplo para explicar isso melhor:

Suponha que você tenha treinado um modelo de deep learning usando Python junto com algumas bibliotecas, como Tensorflow ou JAX. Para isso, você criou um ambiente virtual em sua máquina local. Tudo funciona bem, mas agora você quer compartilhar esse modelo com um colega seu que não tem o Python instalado, muito menos qualquer uma das bibliotecas necessárias.

Em um mundo pré-Docker, seu colega teria que instalar todo esse software apenas para executar seu modelo. Em vez disso, ao instalar o Docker, você pode compartilhar uma imagem do Docker que inclua todo o seu software e isso será tudo o que será necessário.

### Alguns conceitos-chave

Você acabou de ler sobre as imagens do Docker e pode estar se perguntando o que elas são. Agora você será apresentado a três conceitos fundamentais para entender como o Docker funciona. Eles são **Dockerfiles**, **imagens** e **contêineres**, e serão explicados nesta ordem, pois cada um deles usa os anteriores.

- `Dockerfile`: Esse é um arquivo especial que contém todas as instruções necessárias para criar uma imagem. Essas instruções podem ser qualquer coisa, desde "instalar o Python versão 3.7" até "copiar meu código dentro da imagem".

- `Image`: Isso se refere à coleção de todo o seu software em um único local. Usando o exemplo anterior, a imagem incluirá Python, Tensorflow, JAX e seu código. Isso será obtido definindo-se as instruções apropriadas no Dockerfile.

- `Container`: Essa é uma instância em execução de uma imagem. As imagens por si só não fazem muito além de salvar as informações de seu código e suas dependências. É necessário executar um contêiner a partir delas para realmente executar o código dentro delas. Em geral, os contêineres são destinados a executar uma única tarefa, mas podem ser usados como runtime para executar softwares que você não instalou.

Agora que você tem uma ideia de alto nível de como o Docker funciona, é hora de instalá-lo. Se já o tiver instalado, pode pular a maioria dos itens a seguir.

------
## Instalação

Para instalar a versão gratuita do Docker, acesse este [link](https://www.docker.com/products/docker-desktop). 

----
## Testando a instalação do Docker

Para testar sua instalação, vá para a linha de comando e digite o seguinte comando:
```bash
docker run hello-world
```
Esse comando rodará a imagem docker `_/hello-world`. Como você provavelmente não tem essa imagem instalada, o Docker a procurará no [Docker Hub](https://hub.docker.com/) e a levará para sua máquina para executá-la. Se tudo tiver funcionado bem, você verá a seguinte imagem impressa em sua linha de comando:
```bash
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```
-----
### Verifique sua instalação do curl

Outra ferramenta que você usará durante estes laboratórios é [curl](https://curl.se/), portanto, verifique se ela está instalada. Para isso, use o seguinte comando:
```bash
curl -V
```

Se o comando `curl` não for reconhecido, você poderá instalá-lo em um computador Linux baseado no Debian (como o Ubuntu) com este comando:

```bash
apt-get install curl
```

Agora você está pronto para seguir em frente com o restante dos laboratórios! **Bom trabalho!**