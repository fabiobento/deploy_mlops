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

### Configure a inicialização do servidor

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

Agora que a imagem foi criada com sucesso, é hora de executar um contêiner a partir dela. Você pode fazer isso usando o seguinte comando:

```bash
docker run --rm -p 80:80 mlepc4w2-ugl:no-batch
```

Vamos fazer uma rápida recapitulação dos sinalizadores(*flags*) usados na linha de comando acima:
- `--rm`: Exclui esse contêiner depois de parar de executá-lo. Isso evita a necessidade de excluir o contêiner manualmente. A exclusão de contêineres não utilizados ajuda o sistema a ficar limpo e organizado.
- `-p 80:80`: Esse sinalizador executa uma operação conhecida como mapeamento de porta. O contêiner, assim como sua máquina local, tem seu próprio conjunto de portas. Para que você possa acessar a porta 80 dentro do contêiner, é necessário mapeá-la para uma porta no seu computador. Nesse caso, ela é mapeada para a porta 80 em seu computador. 

No final do comando, há o nome e a tag da imagem que você deseja executar. 

Após alguns segundos, o contêiner será iniciado e o servidor será ativado. Você poderá ver os logs do FastAPI sendo impressos no terminal. 

Agora, acesse [localhost:80](http://localhost:80) e você verá uma mensagem sobre a inicialização correta do servidor.

**Bom trabalho!**

## Faça solicitações ao servidor

Agora que o servidor está ouvindo solicitações na porta 80, você pode enviar solicitações `POST` a ele para prever classes de vinho. 

Cada solicitação deve conter os dados que representam um vinho no formato `JSON` como este:

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

Este exemplo representa um vinho de classe 1.

Lembre-se de que o FastAPI tem um cliente interno para você interagir com o servidor. Você pode usá-lo visitando [localhost:80/docs](http://localhost:80/docs)

Também é possível usar o `curl` e enviar os dados diretamente com a solicitação desta forma (observe que é necessário abrir uma nova janela de terminal para isso, pois a que você usou originalmente para ativar o servidor está registrando informações e não poderá ser usada até que você a interrompa):

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

Ou você pode usar um arquivo `JSON` para evitar digitar um comando longo como este:

```bash
curl -X POST http://localhost:80/predict \
    -d @./wine-examples/1.json \
    -H "Content-Type: application/json"
```

Vamos entender os flags usados:
- `-X`: Permite que você especifique o tipo de solicitação. Nesse caso, trata-se de uma solicitação `POST`.
- `-d`: Significa `data` e permite que você anexe dados à solicitação.
- `-H`: Significa `Headers` e permite que você passe informações adicionais por meio da solicitação. Nesse caso, ele é usado para informar ao servidor que os dados são enviados em um formato `JSON`.

Há um diretório chamado `wine-examples` que inclui três arquivos, um para cada classe de vinho. Use-os para testar o servidor e também passe alguns valores aleatórios para ver o que você obtém!

----
**Parabéns por concluir esse laboratório!**

Durante este laboratório, você viu como codificar um servidor da Web usando a FastAPI e como colocá-lo no Docker. Você também aprendeu como `Dockerfiles`, `images` e `containers` estão relacionados entre si e como usar `curl` para fazer solicitações `POST` para obter previsões de servidores.

Na próxima parte, você modificará o servidor para aceitar lotes de dados em vez de um único ponto de dados por solicitação.

Vamos agora para [Parte 2 - Adicionando lotes ao servidor](../with-batch/README.md).
