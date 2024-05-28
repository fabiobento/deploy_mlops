# Implantar um modelo de ML com fastAPI e Docker

Durante este laboratório você implantará um servidor da Web que hospeda um modelo preditivo treinado no [wine dataset](https://scikit-learn.org/stable/modules/generated/sklearn.datasets.load_wine.html#sklearn.datasets.load_wine) usando [FastApi](https://fastapi.tiangolo.com/) e [Docker](https://www.docker.com/).

Neste laboratório você um modelo ao Docker para que ele seja portátil e possa ser implantado com mais facilidade.

Observe que, neste laboratório, você usará um modelo classificador simples que consiste em um  [StandardScaler](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.StandardScaler.html) e um [Random Forest](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html) com florestas de 10 árvores. Utilizaremos esse modelo simples para evitar o uso de bibliotecas como o Tensorflow, que, devido ao seu tamanho, produzirá tempos de compilação muito mais longos ao criar a imagem do Docker.


Abra seu terminal linux e vamos começar!

----
## Por que não usar o Tensorflow Serving??

Nem todos os modelos com os quais você trabalhará ao longo de sua carreira serão escritos no Tensorflow, e eles podem nem mesmo ser modelos de aprendizagem profunda. O TFS é uma ótima opção ao trabalhar com o Tensorflow, mas, na maioria das vezes, você precisará da flexibilidade extra que vem com a codificação de um servidor Web por conta própria.

Além disso, é uma ótima experiência de aprendizado para entender melhor como os servidores da Web se integram aos classificadores para implantar algoritmos de ML.

----


## Como esse laboratório funciona

Ao final deste laboratório, você terá criado duas versões do servidor da Web, uma que pode gerar apenas uma previsão por solicitação e outra que permite a criação de lotes. Durante esse processo, você aprenderá sobre alguns dos recursos da FastAPI, como criar um `Dockerfile` e outros conceitos importantes do Docker, como `image tagging`.

A melhor maneira de acompanhar o desenvolvimento do laboratório é ler esta documentação em seu navegador enquanto trabalha em uma versão clonada deste repositório em seu computador local usando um terminal.

Na documentação, serão exibidos trechos dos arquivos com uma descrição do que está acontecendo. Observe que o mesmo código pode ser encontrado no repositório.

Para clonar esse repositório, use o seguinte comando:

```bash
git clone https://github.com/https-deeplearning-ai/machine-learning-engineering-for-production-public.git
```

ou para clonar via SSh use:

```bash
git clone git@github.com:https-deeplearning-ai/machine-learning-engineering-for-production-public.git
```

--------

Vamos começar pela seção [Parte 1 - uma predição por solicitação](./no-batch/README.md)!

Ou se você já comletou a parte 1 você pode ir direto para a [Parte 2 - Adição de lotes ao servidor](./with-batch/README.md)!
