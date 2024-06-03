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

## Dockerizando o servidor

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


## Criar o Dockerfile

O `Dockerfile` é composto de todas as instruções necessárias para criar sua imagem. Se esta é a primeira vez que você vê esse tipo de arquivo, ele pode parecer intimidador, mas você verá que, na verdade, é mais fácil do que parece. Primeiro, dê uma olhada no arquivo inteiro:

### Base Image
```Dockerfile
FROM frolvlad/alpine-miniconda3:python3.7
```

The `FROM` instruction allows you to select a pre-existing image as the base for your new image. **This means that all of the software available in the base image will also be available on your own.** This is one of Docker's nicest features since it allows for reusing images when needed. 

In this case your base image is `frolvlad/alpine-miniconda3:python3.7`, let's break it down:

- `frolvlad` is the username of the author of the image.
- `alpine-miniconda3` is its name.
- `python3.7` is the image's tag.

This image contains an [alpine](https://alpinelinux.org/) version of Linux, which is a distribution created to be very small in size. It also includes [miniconda](https://docs.conda.io/en/latest/miniconda.html) with Python 3. Notice that the tag let's you know that the specific version of Python being used is 3.7. Tagging is great as it allows you to create different versions of similar images. In this case you could have this same image with a different version of Python such as 3.5.

You could use many different base images such as the official `python:3.7` image. However if you compared the size you will encounter it is a lot heavier. In this case you will be using the one mentioned above as it is a great minimal image for the task at hand.

### Installing dependencies
Now that you have an environment with Python installed it is time to install all of the Python packages that your server will depend on. First you need to copy your local `requirements.txt` file into the image so it can be accessed by other processes, this can be done via the `COPY` instruction:

```Dockerfile
COPY requirements.txt .
```

Now you can use `pip` to install these Python libraries. To run any command as you would on `bash`, use the `RUN` instruction:
```Dockerfile
RUN pip install -r requirements.txt && \
	rm requirements.txt
```
Notice that two commands were chained together using the `&&` operator. After you installed the libraries specified within `requirements.txt` you don't have more use for that file so it is a good idea to delete it so the image includes only the necessary files for your server to run.

This can be done using two `RUN` instructions, however, it is a good practice to chain together commands in this manner since Docker creates a new layer every time it encounters a `RUN`, `COPY` or `ADD` instruction. This will result in a bigger image size. If you are interested in best practices for writing Dockerfiles be sure to check out this [resource](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/).


### Exposing the port

Since you are coding a web server it is a good idea to leave some documentation about the port that the server is going to listen on. You can do this with the `EXPOSE` instruction. In this case the server will listen to requests on port 80: 

```Dockerfile
EXPOSE 80
```

### Copying your server into the image

Now you should put your code within the image. To do this you can simply use the `COPY` instruction to copy the `app` directory within the root of the container:

```Dockerfile
COPY ./app /app
```

### Spinning up the server

Containers are usually meant to start and carry out a single task. This is why the `CMD` instruction was created. This is the command that will be run once a container that uses this image is started. In this case it is the command that will spin up the server by specifying the host and port. Notice that the command is written in a `JSON` like format having each part of the command as a string within a list:

```Dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```

What is meant by `JSON` like format is that Docker uses `JSON` for its configurations and the `CMD` instruction expects the commands as a list that follows `JSON` conventions.

### Putting it all together

The resulting `Dockerfile` will look like this:

```Dockerfile
FROM frolvlad/alpine-miniconda3:python3.7

COPY requirements.txt .

RUN pip install -r requirements.txt && \
	rm requirements.txt

EXPOSE 80

COPY ./app /app

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "80"]
```

Remember you can also look at it within the `no-batch` directory.


## Build the image

Now that the `Dockerfile` is ready and you understand its contents, it is time to build the image. To do so, double check that you are within the `no-batch` directory and use the `docker build` command.
```bash
docker build -t mlepc4w2-ugl:no-batch .
```

You can use the `-t` flag to specify the name of the image and its tag. As you saw earlier the tag comes after the colon so in this case the name is `mlepc4w2-ugl` and the tag is `no-batch`.

After a couple of minutes your image should be ready to be used! If you want to see it along with any other images that you have on your local machine use the `docker images` command. This will display all of you images alongside their names, tags and size.



## Run the container

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
