Adaptado de [Machine Learning in Production](https://www.deeplearning.ai/courses/machine-learning-in-production/) de [Andrew Ng](https://www.deeplearning.ai/)  ([Stanford University](http://online.stanford.edu/), [DeepLearning.AI](https://www.deeplearning.ai/))

## Adição de lotes ao servidor

Agora que você viu como codificar um servidor da Web que serve um modelo para inferência, é hora de implementar algum código que permita obter previsões em lote. Isso é muito importante porque, no momento, seu servidor só pode lidar com uma previsão por solicitação e você está perdendo esse recurso para o qual a maioria dos preditores de aprendizado de máquina foi otimizada.

Inicie utilizando o comando `cd` para entrar no diretório `with-batch`. Se você estiver atualmente no diretório `no-batch`, poderá usar o comando `cd ../with-batch`.

## Aualize o servidor

Lembre-se de que esse código pode ser encontrado no arquivo `app/main.py`.

Em primeiro lugar, você precisará de duas importações adicionais para lidar com a previsão por lotes. Ambas estão relacionadas à manipulação de dados do tipo lista. São elas `List` de `typing` e `conlist` de `pydantic`. Lembre-se de que o `REST` não oferece suporte a objetos como arrays numpy, portanto, você precisa serializar esse tipo de dados em listas. As importações agora terão a seguinte aparência:
```python
import pickle
import numpy as np
from typing import List
from fastapi import FastAPI
from pydantic import BaseModel, conlist
```

Agora você modificará a classe `Wine`. Ela costumava representar um vinho, mas agora representará um lote de vinhos. Para fazer isso, você pode definir o atributo `batches` e especificar que ele será do tipo `List` de `conlist`. Como a FastAPI impõe tipos de objetos, você precisa especificá-los explicitamente. Nesse caso, você sabe que o lote será uma lista de tamanho arbitrário, mas também precisa especificar o tipo dos elementos dessa lista. Você poderia fazer uma lista de listas de floats, mas há uma alternativa melhor, usando o conlist do pydantic. O prefixo “con” significa `constrained`, portanto, essa é uma lista restrita. Esse tipo permite que você selecione o tipo de itens dentro da lista e também o número máximo e mínimo de itens. Nesse caso, seu modelo foi treinado usando 13 características, portanto, cada ponto de dados deve ter o tamanho 13:


```python
# Representa um lote de vinhos
class Wine(BaseModel):
    batches: List[conlist(item_type=float, min_length=13, max_length=13)]
```

Observe que você não está nomeando explicitamente cada característica, portanto, nesse caso, a **ordem dos dados é importante**.

Por fim, você atualizará o endpoint `predict`, pois o carregamento do classificador é o mesmo que no caso sem lotes. 

Como o scikit-learn aceita lotes de dados representados por matrizes numpy, você pode simplesmente obter os lotes do objeto `wine`, que são listas, e convertê-los em matrizes numpy antes de alimentá-los no classificador. Mais uma vez, você precisa converter as previsões em uma lista para torná-las compatíveis com REST:

```python
@app.post("/predict")
def predict(wine: Wine):
    batches = wine.batches
    np_batches = np.array(batches)
    pred = clf.predict(np_batches).tolist()
    return {"Prediction": pred}
```

Com essas pequenas alterações, seu servidor agora está pronto para aceitar lotes de dados! Massa!

## Criação de uma nova versão da imagem

O Dockerfile para essa nova versão do servidor é idêntico ao anterior. As únicas alterações foram feitas no arquivo `main.py`, portanto, o Dockerfile permanece o mesmo.

Agora, enquanto estiver no diretório `with-batch`, execute o comando docker build:

```bash
docker build -t mlepc4w2-ugl:with-batch . 
```

Como as camadas iniciais são idênticas às da imagem anterior, o processo de compilação deve ser bastante rápido. Esse é outro recurso excelente do Docker chamado `layer caching`, que armazena em cache as camadas para compilações mais recentes a fim de reduzir o tempo de compilação.

Observe que o nome da imagem permanece o mesmo, mas a tag foi alterada para especificar que essa versão do servidor oferece suporte a lotes.

## Limpando as coisas

Para verificar as duas imagens que você criou, use o seguinte comando:

```bash
docker images
```

Às vezes, ao verificar todas as imagens disponíveis, você se deparará com algumas imagens com nomes e tags que têm o valor `none`. Essas são imagens intermediárias e, se você as vir listadas ao usar o comando `docker images`, geralmente é uma boa prática removê-las. 


Outra maneira de verificar se você tem essas imagens em seu sistema é executar este comando:

```bash
$(docker images --filter "dangling=true" -q --no-trunc)
```

Se não houver saída, significa que não há imagens desse tipo. Caso existam, você pode removê-las executando o seguinte comando:

```bash
docker rmi $(docker images --filter "dangling=true" -q --no-trunc)
```

Agora, se você executar novamente o comando `docker images`, não verá essas imagens intermediárias. As coisas estão muito mais limpas agora!

## Rodar o servidor

Agora, você pode executar um contêiner a partir da imagem usando o seguinte comando:

```bash
docker run --rm -p 81:80 mlepc4w2-ugl:with-batch 
```
Observe que, desta vez, a porta 80 no contêiner é mapeada para a porta 81 no seu host local. Isso ocorre porque você provavelmente não interrompeu o servidor da parte 1 deste laboratório, que ainda está em execução na porta 80. Isso também serve para mostrar como o mapeamento de portas pode ser usado para mapear qualquer porta no contêiner para qualquer porta no host.

Agora, vá até [localhost:81] (http://localhost:81) e você verá uma mensagem sobre o servidor estar funcionando corretamente.


##Faça uma solicitação ao servidor

Mais uma vez, é hora de testar seu servidor usando-o de fato para fazer previsões. Da mesma forma que antes, há alguns exemplos de lotes de dados no diretório `wine-examples`. 

Para obter as previsões de um lote de 32 vinhos (encontradas no arquivo batch_1.json), você pode enviar uma solicitação `POST` para o servidor usando `curl` como esta:

```bash
curl -X POST http://localhost:81/predict \
    -d @./wine-examples/batch_1.json \
    -H "Content-Type: application/json"
```

Agora você deve ver uma lista com as 32 previsões (em ordem) para cada um dos pontos de dados dentro do lote. **Muito bom!**

## Parar o servidor

Para acessar os servidores e os contêineres em que eles estão sendo executados, basta usar a combinação de teclas `ctrl + c` na janela do terminal em que o processo foi iniciado.

Como alternativa, você pode usar o comando `docker ps` para verificar o nome dos contêineres em execução e usar o comando `docker stop name_of_container` para interrompê-los. Lembre-se de que você usou o sinalizador `--rm`, portanto, assim que parar esses contêineres, eles também serão excluídos.

**Não exclua as imagens que você criou, pois elas serão usadas em um laboratório futuro!**
-----

**Parabéns por terminar este laboratório!**

Agora você deve ter uma melhor compreensão de como os servidores da Web podem ser usados para hospedar seus modelos de aprendizado de máquina. Você viu como pode usar uma biblioteca como a FastAPI para codificar o servidor e usar o Docker para enviar o servidor junto com o modelo de maneira fácil. Você também aprendeu sobre alguns conceitos-chave do Docker, como `image tagging`(marcação de imagens) e `port mapping`(mapeamento de portas), e como permitir a criação de lotes nas solicitações. Em geral, você deve ter uma ideia mais clara de como todas essas tecnologias interagem para hospedar modelos em produção.

**Continue assim!**

Vamos agora para o laboratório [Introdução ao Kubernetes](../../C4_W2_Lab_2_Intro_to_Kubernetes/README.md).