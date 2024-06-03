Adaptado de [Machine Learning in Production](https://www.deeplearning.ai/courses/machine-learning-in-production/) de [Andrew Ng](https://www.deeplearning.ai/)  ([Stanford University](http://online.stanford.edu/), [DeepLearning.AI](https://www.deeplearning.ai/))

## Uma previsão por solicitação
Você pode encontrar todo o código a seguir no diretório `no-batch/`. Em seu terminal, digite `cd` nesse diretório para continuar com o laboratório. Se você estiver no momento na raiz do repositório, poderá usar o comando `cd 2\ -\ Servir\ modelo-padrões\ de\ infraestrutura/week2-ungraded-labs/C4_W2_Lab_1_FastAPI_Docker/no-batch/`.

Observe que o código do servidor deve estar no arquivo `main.py` em um diretório chamado `app`, seguindo as diretrizes da FastAPI.

### Codificação do servidor

Comece importando as dependências necessárias. Você usará o `pickle` para carregar o modelo pré-treinado salvo no arquivo `app/wine.pkl`, o `numpy` para manipulação de tensores e o restante para desenvolver o servidor Web com o `FastAPI`.

Além disso, crie uma instância da classe `FastAPI`. Essa instância tratará de todas as funcionalidades do servidor:

```python
import pickle
import numpy as np
from fastapi import FastAPI
from pydantic import BaseModel


app = FastAPI(title="Predicting Wine Class")
```

Agora você precisa de uma maneira de representar um ponto de dados. Para isso, crie uma classe que seja subclasse do `BaseModel` da pydantic e liste cada atributo com seu tipo correspondente

Nesse caso, um ponto de dados representa um vinho, portanto, essa classe é chamada de `Wine` e todos os recursos do modelo são do tipo `float`:

```python

# Representa um determinado vinho (ou ponto de dados)
class Wine(BaseModel):
class Wine(BaseModel):
    alcohol: float
    malic_acid: float
    ash: float
    alcalinity_of_ash: float
    magnesium: float
    total_phenols: float
    flavanoids: float
    nonflavanoid_phenols: float
    proanthocyanins: float
    color_intensity: float
    hue: float
    od280_od315_of_diluted_wines: float
    proline: float
```

Agora é hora de carregar o classificador na memória para que ele possa ser usado para a previsão. Isso pode ser feito no escopo global do script, mas aqui é feito dentro de uma função para mostrar a você um recurso interessante da FastAPI.

Se você decorar uma função com o decorador `@app.on_event(“startup”)`, garantirá que a função seja executada na inicialização do servidor. Isso lhe dá alguma flexibilidade se você precisar que alguma lógica personalizada seja acionada logo quando o servidor for iniciado.

O classificador é aberto usando um gerenciador de contexto e atribuído à variável `clf`, que você ainda precisa tornar global para que outras funções possam acessá-la:


```python

@app.on_event("startup")
def load_clf():
    # Carregar classificador do arquivo pickle
    with open("/app/wine.pkl", "rb") as file:
        global clf
        clf = pickle.load(file)
```

Por fim, você precisa criar a função que tratará da previsão. Essa função será executada quando você visitar o *endpoint* `/predict` do servidor e esperar um ponto de dados `Wine`.

Essa função é, na verdade, muito simples: primeiro você converterá as informações do objeto `Wine` em uma matriz numpy de formato `(1, 13)` e, em seguida, usará o método `predict` do classificador para fazer uma previsão para o ponto de dados. Observe que a previsão deve ser convertida em uma lista usando o método `tolist`.

Por fim, retorne um dicionário (que a FastAPI converterá em `JSON`) contendo a previsão.


```python

@app.post("/predict")
def predict(wine: Wine):
    data_point = np.array(
        [
            [
                wine.alcohol,
                wine.malic_acid,
                wine.ash,
                wine.alcalinity_of_ash,
                wine.magnesium,
                wine.total_phenols,
                wine.flavanoids,
                wine.nonflavanoid_phenols,
                wine.proanthocyanins,
                wine.color_intensity,
                wine.hue,
                wine.od280_od315_of_diluted_wines,
                wine.proline,
            ]
        ]
    )

    pred = clf.predict(data_point).tolist()
    pred = pred[0]
    print(pred)
    return {"Predição": pred}

```

Agora, o código do servidor está pronto para inferência, embora você ainda precise ativá-lo. Se quiser testá-lo localmente (desde que tenha as dependências necessárias instaladas), você pode fazer isso usando o comando `uvicorn main:app --reload` no mesmo diretório do arquivo `main.py`. **No entanto, isso não é necessário, pois você fará a "dockerização" desse servidor em seguida**.

## "Dockerize" o servidor

A partir de agora, todos os comandos serão executados presumindo-se que você esteja no diretório `no-batch/`.

Além disso, você deve criar um diretório chamado `app` e colocar o `main.py` (o servidor) e suas dependências (`wine.pkl`) lá, conforme explicado na [documentação oficial do FastAPI](https://fastapi.tiangolo.com/deployment/docker/) sobre como implantar com o Docker. Isso deve resultar em uma estrutura de diretório semelhante a esta:

```
..
└── no-batch
    ├── app/
    │   ├── main.py (código do servidor)
    │   └── wine.pkl (classificador serializado)
    ├── requirements.txt (Dependências python)
    ├── wine-examples/ (Exemplos do dataset wine para testar o servidor)
    ├── README.md (esse arquivo)
    └── Dockerfile
```


## Crie o Dockerfile

O `Dockerfile` é composto de todas as instruções necessárias para criar sua imagem. Se esta é a primeira vez que você vê esse tipo de arquivo, ele pode parecer intimidador, mas você verá que, na verdade, é mais fácil do que parece. Primeiro, dê uma olhada no arquivo inteiro:

### Imagem base
```Dockerfile
FROM frolvlad/alpine-miniconda3:python3.7
```

A instrução `FROM` permite que você selecione uma imagem pré-existente como base para sua nova imagem. **Isso significa que todo o software disponível na imagem base também estará disponível na sua própria imagem.** Esse é um dos recursos mais interessantes do Docker, pois permite a reutilização de imagens quando necessário.

Nesse caso, sua imagem de base é `frolvlad/alpine-miniconda3:python3.7`. Vamos ententendê-la passo-a-passo:

- `frolvlad` é o nome de usuário do autor da imagem.
- `alpine-miniconda3` é o nome da imagem.
- `python3.7` é a *tag* da imagem.

Essa imagem contém uma versão [alpine](https://alpinelinux.org/) do Linux, que é uma distribuição criada para ser muito pequena em tamanho. Ela também inclui a [miniconda](https://docs.conda.io/en/latest/miniconda.html) com Python 3. Observe que a tag permite que você saiba que a versão específica do Python que está sendo usada é a 3.7. A marcação com tag é excelente, pois permite que você crie versões diferentes de imagens semelhantes. Nesse caso, você poderia ter essa mesma imagem com uma versão diferente do Python, como a 3.5.

Você poderia usar muitas imagens de base diferentes, como a imagem oficial `python:3.7`. Entretanto, se você comparar o tamanho, verá que ela é muito mais pesada. Nesse caso, você usará a imagem mencionada acima, pois ela é uma imagem mínima excelente para a tarefa em questão.

### Instale as dependências
Agora que você tem um ambiente com o Python instalado, é hora de instalar todos os pacotes Python dos quais seu servidor dependerá. Primeiro, é necessário copiar o arquivo local `requirements.txt` para a imagem, de modo que ele possa ser acessado por outros processos, o que pode ser feito por meio da instrução `COPY`:

```Dockerfile
COPY requirements.txt .
```

Agora você pode usar o `pip` para instalar essas bibliotecas Python. Para executar qualquer comando como você faria no `bash`, use a instrução `RUN`:

```Dockerfile
RUN pip install -r requirements.txt && \
	rm requirements.txt
```

Observe que dois comandos foram encadeados usando o operador `&&`. Depois de instalar as bibliotecas especificadas em `requirements.txt`, você não terá mais utilidade para esse arquivo, portanto, é uma boa ideia excluí-lo para que a imagem inclua apenas os arquivos necessários para a execução do servidor.

Isso pode ser feito usando duas instruções `RUN`; no entanto, é uma boa prática encadear comandos dessa maneira, pois o Docker cria uma nova camada toda vez que encontra uma instrução `RUN`, `COPY` ou `ADD`. Isso resultará em um tamanho de imagem maior. Se você estiver interessado nas práticas recomendadas para escrever Dockerfiles, não deixe de conferir esta [Visão geral das práticas recomendadas para escrever Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).


### Exponha a porta

Como você está programando um servidor da Web, é uma boa ideia deixar alguma documentação sobre a porta em que o servidor vai escutar. Você pode fazer isso com a instrução `EXPOSE`. Nesse caso, o servidor escutará as solicitações na porta 80:  

```Dockerfile
EXPOSE 80
```

### Copie seu servidor para a imagem

Agora você deve colocar seu código dentro da imagem. Para fazer isso, basta usar a instrução `COPY` para copiar o diretório `app` na raiz do contêiner:

```Dockerfile
COPY ./app /app
```

### Coloque o servidor pra funcionar

Em geral, os contêineres são destinados a iniciar e executar uma única tarefa. É por isso que a instrução `CMD` foi criada. Esse é o comando que será executado quando um contêiner que usa essa imagem for iniciado. Nesse caso, é o comando que ativará o servidor especificando o host e a porta. Observe que o comando é escrito em um formato semelhante ao `JSON`, com cada parte do comando como uma string em uma lista:

```Dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```

Isso é conhecido como formato semelhante ao `JSON`(*JSON like*), e o Docker usa o `JSON` para suas configurações e a instrução `CMD` espera os comandos como uma lista que segue as convenções do `JSON`.

### "Juntando" tudo

O `Dockerfile` resultante terá a seguinte aparência:

```Dockerfile
FROM frolvlad/alpine-miniconda3:python3.7

COPY requirements.txt .

RUN pip install -r requirements.txt && \
	rm requirements.txt

EXPOSE 80

COPY ./app /app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```

Lembre-se de que você também pode examiná-lo no diretório `no-batch`.


## Crie a imagem

Agora que o `Dockerfile` está pronto e você entendeu seu conteúdo, é hora de criar a imagem. Para fazer isso, verifique se você está no diretório `no-batch` e use o comando `docker build`.

```bash
docker build -t mlepc4w2-ugl:no-batch .
```

Você pode usar o sinalizador `-t` para especificar o nome da imagem e sua tag. Como você viu anteriormente, a tag vem após os dois pontos, portanto, nesse caso, o nome é `mlepc4w2-ugl` e a tag é `no-batch`.

Após alguns minutos, sua imagem deverá estar pronta para ser usada! Se quiser vê-la junto com qualquer outra imagem que tenha em sua máquina local, use o comando `docker images`. Isso exibirá todas as suas imagens juntamente com seus nomes, tags e tamanho.

## Execute o container

Now that the image has been successfully built it is time to run a container out of it. You can do so by using the following command:

```bash
docker run --rm -p 80:80 mlepc4w2-ugl:no-batch
```

You should recognize this command from a previous ungraded lab. Let's do a quick recap of the flags used:
- `--rm`: Delete this container after stopping running it. This is to avoid having to manually delete the container. Deleting unused containers helps your system to stay clean and tidy.
- `-p 80:80`: This flags performs an operation knows as port mapping. The container, as well as your local machine, has its own set of ports. So you are able to access the port 80 within the container, you need to map it to a port on your computer. In this case it is mapped to the port 80 in your machine. 

At the end of the command is the name and tag of the image you want to run. 

After some seconds the container will start and spin up the server within. You should be able to see FastAPI's logs being printed in the terminal. 

Now head over to [localhost:80](http://localhost:80) and you should see a message about the server spinning up correctly.

**Nice work!**

## Make requests to the server

Now that the server is listening to requests on port 80, you can send `POST` requests to it for predicting classes of wine. 

Every request should contain the data that represents a wine in `JSON` format like this:

```json
{
  "alcohol":12.6,
  "malic_acid":1.34,
  "ash":1.9,
  "alcalinity_of_ash":18.5,
  "magnesium":88.0,
  "total_phenols":1.45,
  "flavanoids":1.36,
  "nonflavanoid_phenols":0.29,
  "proanthocyanins":1.35,
  "color_intensity":2.45,
  "hue":1.04,
  "od280_od315_of_diluted_wines":2.77,
  "proline":562.0
}
```

This example represents a class 1 wine.

Remember from Course 1 that FastAPI has a built-in client for you to interact with the server. You can use it by visiting [localhost:80/docs](http://localhost:80/docs)

You can also use `curl` and send the data directly with the request like this (notice that you need to open a new terminal window for this as the one you originally used to spin up the server is logging info and is not usable until you stop it):

```bash
curl -X 'POST' http://localhost/predict \
  -H 'Content-Type: application/json' \
  -d '{
  "alcohol":12.6,
  "malic_acid":1.34,
  "ash":1.9,
  "alcalinity_of_ash":18.5,
  "magnesium":88.0,
  "total_phenols":1.45,
  "flavanoids":1.36,
  "nonflavanoid_phenols":0.29,
  "proanthocyanins":1.35,
  "color_intensity":2.45,
  "hue":1.04,
  "od280_od315_of_diluted_wines":2.77,
  "proline":562.0
}'
```

Or you can use a `JSON` file to avoid typing a long command like this:

```bash
curl -X POST http://localhost:80/predict \
    -d @./wine-examples/1.json \
    -H "Content-Type: application/json"
```

Let's understand the flags used:
- `-X`: Allows you to specify the request type. In this case it is a `POST` request.
- `-d`: Stands for `data` and allows you to attach data to the request.
- `-H`: Stands for `Headers` and it allows you to pass additional information through the request. In this case it is used to the tell the server that the data is sent in a `JSON` format.


There is a directory called `wine-examples` that includes three files, one for each class of wine. Use those to try out the server and also pass in some random values to see what you get!

----
**Congratulations on finishing part 1 of this ungraded lab!**

During this lab you saw how to code a webserver using FastAPI and how to Dockerize it. You also learned how `Dockerfiles`, `images` and `containers` are related to each other and how to use `curl` to make `POST` requests to get predictions from servers.

In the next part you will modify the server to accept batches of data instead of a single data point per request.

Jump to [Part 2 - Adding batching to the server](../with-batch/README.md)
