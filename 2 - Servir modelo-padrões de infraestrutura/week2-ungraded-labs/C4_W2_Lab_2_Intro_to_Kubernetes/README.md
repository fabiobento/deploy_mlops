Adaptado de [Machine Learning in Production](https://www.deeplearning.ai/courses/machine-learning-in-production/) de [Andrew Ng](https://www.deeplearning.ai/)  ([Stanford University](http://online.stanford.edu/), [DeepLearning.AI](https://www.deeplearning.ai/))

# Introdução a Kubernetes
Nesse laboratório utilizaremos o Kubernetes e, antes de prosseguir, recomendo que você dê uma olhada em como ele funciona. O site é
https://kubernetes.io/
e, na parte superior da página, há um grande botão que diz "*Learn Kubernetes Basics*" (Aprenda o básico do Kubernetes).
<img src='img/introk8s.png ' alt='img/kubernetes.png'>

Clique nele e você será direcionado para: 
https://kubernetes.io/docs/tutorials/kubernetes-basics/

A partir daí, você pode passar por um tutorial de como criar um cluster, implantar um aplicativo, dimensioná-lo, atualizá-lo e muito mais. É interativo, divertido e vale a pena dedicar algumas horas do seu tempo para realmente entender como o Kubernetes funciona.

Talvez você também queira conferir este [tutorial em vídeo](https://youtu.be/H06qrNmGqyE). 

# Pratique o Kubernetes em seu ambiente local

No item anterior, você viu os conceitos básicos do Kubernetes executando comandos imperativos no shell interativo (ou seja, com o `kubectl`). Ele mostra como essa ferramenta pode ser usada para orquestrar contêineres para hospedar um aplicativo. Se você não tiver feito esse exercício, recomendamos que volte a ele, pois presumiremos que você já esteja familiarizado com os conceitos discutidos [naqueles 6 módulos](https://kubernetes.io/docs/tutorials/kubernetes-basics/). Este laboratório será uma extensão dessa atividade e mostrará mais alguns conceitos. Especificamente, você irá:

* configurar o Kubernetes em seu computador local para aprendizado e desenvolvimento
* create Kubernetes objects using YAML files
* criar objetos do Kubernetes usando arquivos YAML
* acessar a implantação usando um serviço Nodeport
* dimensionar automaticamente a implantação para lidar dinamicamente com o tráfego de entrada 

## Instalação

Primeiro, você configurará sua máquina para executar um cluster local do Kubernetes. Isso lhe dá mais liberdade para ajustar as configurações e implementar mais recursos em comparação com o ambiente Katacoda on-line que você usou no tutorial do Kubernetes. É uma ótima ferramenta para aprendizado e também para desenvolvimento local. Há várias distribuições do Kubernetes e a mais adequada para nosso objetivo é a [Minikube](https://minikube.sigs.k8s.io/docs/). Você deve se lembrar que ela foi mencionada no primeiro módulo do exercício básico do Kubernetes.

Você precisará instalar as seguintes ferramentas para realizar este laboratório:

* **curl** - uma ferramenta de linha de comando para transferência de dados usando vários protocolos de rede. É possível que você já tenha instalado isso anteriormente, mas, caso não tenha, [aqui está uma referência](https://reqbin.com/Article/InstallCurl) para fazer isso. Você usará isso para consultar seu modelo mais tarde.

* **Docker** - O Minikube foi projetado para ser executado virtualizado em uma VM ou container. Por isso vamos utilizar o [Docker](https://docs.docker.com/engine/install/) 18.09 or higher (20.10 ou superior é o remcomentado).

* **kubectl** - a ferramenta de linha de comando para interagir com clusters do Kubernetes. As instruções de instalação podem ser encontradas [aqui](https://kubernetes.io/docs/tasks/tools/).

* **Minikube** - uma distribuição do Kubernetes voltada para novos usuários e trabalho de desenvolvimento. No entanto, ela não se destina a implementações de produção, pois só pode executar um cluster de nó único em sua máquina. Instruções de instalação [aqui](https://minikube.sigs.k8s.io/docs/start/).

## Arquitetura
A aplicação que você criará será parecida com a figura abaixo

<img src='img/kubernetes.png' alt='img/kubernetes.png'>

Você criará uma implantação que ativa contêineres que executam um servidor de modelo. Neste caso, ele será da imagem `tensorflow/serving`. A implantação pode ser acessada por terminais externos (ou seja, seus usuários) por meio de um serviço exposto. Isso traz solicitações de inferência para os servidores de modelo e responde com previsões do seu modelo.

Por fim, a implantação ativará ou desativará os pods com base na utilização da CPU. Ele começará com um pod, mas quando a carga exceder um ponto predefinido, ele ativará pods adicionais para compartilhar a carga.

## Iniciar o Minikube

Agora você está quase pronto para iniciar seu cluster do Kubernetes. Há apenas mais uma etapa adicional. Como mencionado anteriormente, o Minikube é executado dentro de uma máquina virtual. Isso significa que os pods que você criará posteriormente só verão os volumes dentro dessa VM. Portanto, se você quiser carregar um modelo em seus pods, deverá primeiro montar o local desse modelo dentro da VM do Minikube. Vamos configurar isso agora.

Você usará o modelo `half_plus_two` que viu anteriormente. Você pode copiá-lo para o diretório `/var/tmp`. Você pode usar o comando abaixo:

```
cp -R ./saved_model_half_plus_two_cpu /var/tmp
```
Agora você está pronto para iniciar o Minikube! Execute o comando abaixo para inicializar o Virtualbox e montar a pasta que contém seu arquivo de modelo:
   ```
   minikube start --mount=True --mount-string="/var/tmp:/var/tmp" --vm-driver=docker
  ```


---

</details>
</br>

## Criação de objetos com arquivos YAML

No tutorial básico oficial do Kubernetes, você usou principalmente o `kubectl` para criar objetos como pods, implantações e serviços. Embora isso definitivamente funcione, sua configuração será mais portátil e mais fácil de manter se você configurá-los usando arquivos [YAML](https://yaml.org/spec/1.2/spec.html). Incluí esses arquivos no diretório `yaml` deste laboratório para que você possa ver como eles são criados. A [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/) também documenta os campos compatíveis com cada objeto. Por exemplo, a API para Pods pode ser encontrada [aqui](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/pod-v1/).

Uma maneira de gerar isso quando você não tem um modelo para começar é usar primeiro o comando `kubectl` e, em seguida, usar o sinalizador `-o yaml` para gerar o arquivo YAML para você. Por exemplo, o [kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) mostra que você pode gerar o YAML para um pod executando uma imagem `nginx` com esse comando:

```
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```

Todos os objetos necessários para este laboratório já foram fornecidos e você pode modificá-los posteriormente quando quiser praticar configurações diferentes. Vamos examiná-los um a um nas próximas seções.

### Config Maps

Primeiro, você criará um [config map](https://kubernetes.io/docs/concepts/configuration/configmap/) que define uma variável `MODEL_NAME` e `MODEL_PATH`. Isso é necessário devido ao modo como a imagem `tensorflow/serving` é configurada. Se você observar a última camada do arquivo docker [aqui](https://hub.docker.com/layers/tensorflow/serving/2.6.0/images/sha256-7e831f11c9ef928c09b4064a059066484079ac819991f162a938de0ad4b0fbd5?context=explore), verá que ele executa um script `/usr/bin/tf_serving_entrypoint.sh` quando inicia o contêiner. Esse script contém apenas esse comando:

```
#!/bin/bash 

tensorflow_model_server --port=8500 --rest_api_port=8501 --model_name=${MODEL_NAME} --model_base_path=${MODEL_BASE_PATH}/${MODEL_NAME} "$@"
```

Basicamente, ele inicia o servidor de modelos e usa as variáveis de ambiente `MODEL_BASE_PATH` e `MODEL_NAME` para localizar o modelo. Embora você também possa definir isso explicitamente no arquivo YAML `Deployment`, seria mais organizado tê-lo em um configmap para que você possa conectá-lo posteriormente. Abra `yaml/configmap.yaml` para ver a sintaxe.

Você pode criar o objeto agora usando o `kubectl`, conforme mostrado abaixo. Observe o sinalizador `-f` para especificar um nome de arquivo. Você também pode especificar um diretório, mas faremos isso mais tarde.

```
kubectl apply -f yaml/configmap.yaml
```

Com isso, você deve ser capaz de "obter" e "descrever" o objeto como antes. Por exemplo, `kubectl describe cm tfserving-configs` deve mostrar a você:

```
Name:         tfserving-configs
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
MODEL_PATH:
----
/models/half_plus_two
MODEL_NAME:
----
half_plus_two
Events:  <none>
```

### Criar um Deployment

Agora, você criará a implementação do seu aplicativo. Abra `yaml/deployment.yaml` para ver a especificação desse objeto. Você verá que ele inicia uma réplica, usa `tensorflow/serving` como a imagem do contêiner e define variáveis de ambiente por meio da tag `envFrom`. Ele também expõe a porta `8501` do contêiner porque você enviará solicitações HTTP a ele mais tarde. Ele também define os limites de CPU e memória e monta o volume da VM do Minikube para o contêiner.

Como antes, você pode aplicar esse arquivo para criar o objeto:

```
kubectl apply -f yaml/deployment.yaml
```

A execução de `kubectl get deploy` após cerca de 90 segundos deve mostrar algo como o que está abaixo para informar que a implantação está pronta.

```
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
tf-serving-deployment   1/1     1            1           15s
```

### Expor o deployment através de um serviço

Como você aprendeu anteriormente no tutorial do Kubernetes, será necessário criar um serviço para que seu aplicativo possa ser acessado fora do cluster. Incluímos o `yaml/service.yaml` para isso. Ele define um serviço [NodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) que expõe a porta 30001 do nó. As solicitações enviadas para essa porta serão enviadas para a `targetPort` especificada pelos contêineres, que é `8501`. 

Aplique `kubectl apply -f yaml/service.yaml` e execute `kubectl get svc tf-serving-service`. Você deverá ver algo parecido com isto:

```
NAME                 TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
tf-serving-service   NodePort   10.103.30.4   <none>        8501:30001/TCP   27s
```

Você pode tentar acessar a implantação agora para uma verificação se está tudo funcionando bem. O comando `curl` a seguir enviará uma linha de solicitações de inferência para o serviço Nodeport:

```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST $(minikube ip):30001/v1/models/half_plus_two:predict
```

Se o comando acima não funcionar, você pode executar `minikube ip` primeiro para obter o endereço IP do nó do Minikube. Ele deve retornar um endereço IP local como `192.168.99.102` (se você vir `127.0.0.1`, consulte as seções de solução de problemas abaixo). Você pode então inserir esse endereço no comando acima substituindo a string `$(minikube ip)`. Por exemplo:

```
curl -d '{"instances": [1.0, 2.0, 5.0]}' -X POST http://192.168.99.102:30001/v1/models/half_plus_two:predict
```
Se o comando for bem-sucedido, você verá os resultados retornados pelo modelo:

```
{
    "predictions": [2.5, 3.0, 4.5
    ]
}
```

Excelente! Seu aplicativo foi executado com sucesso e pode ser acessado fora do cluster!

### Horizontal Pod Autoscaler

Uma das grandes vantagens da orquestração de contêineres é que ela permite dimensionar o aplicativo de acordo com as necessidades do usuário. O Kubernetes fornece um [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) para criar ou remover conjuntos de réplicas com base nas métricas observadas. Para fazer isso, o HPA consulta um [Metrics Server](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-metrics-pipeline/#metrics-server) para medir a utilização de recursos, como CPU e memória. O Metrics Server não é iniciado por padrão no Minikube e precisa ser ativado com o seguinte comando:

```
minikube addons enable metrics-server
```


Você deverá ver um prompt dizendo ` The 'metrics-server' addon is enabled
`. Isso inicia uma implantação do `metrics-server` no namespace `kube-system`. Execute o comando abaixo e aguarde até que a implantação esteja pronta.



```
kubectl get deployment metrics-server -n kube-system
```

Você deverá ver algo como:

```
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           76s
```

Com isso, agora você pode criar seu autoscaler aplicando `kubectl apply -f yaml/autoscale.yaml`. Aguarde cerca de um minuto para que ele possa consultar o servidor de métricas. A execução de `kubectl get hpa` deve mostrar: 

```
NAME             REFERENCE                          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
tf-serving-hpa   Deployment/tf-serving-deployment   0%/2%    1         3         1          38s

```

Se estiver mostrando `Unknown` em vez de `0%` na coluna `TARGETS`, você pode tentar enviar alguns comandos curl como fez anteriormente e aguardar mais um minuto.


### Teste de Stress

Para testar o recurso de dimensionamento automático de sua implantação, fornecemos um script bash curto (`request.sh`) que enviará solicitações de forma persistente ao seu aplicativo. Abra uma nova janela de terminal, certifique-se de que está no diretório raiz deste arquivo README e, em seguida, execute este comando:

```
/bin/bash request.sh
```

Você deverá ver os resultados sendo impressos em rápida sucessão:

```
{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{
    "predictions": [2.5, 3.0, 4.5
    ]
}{

```

Se estiver vendo uma conexão recusada, verifique se o serviço ainda está em execução com `kubectl get svc tf-serving-service`.


Há várias maneiras de monitorar isso, mas a mais fácil seria usar o painel de controle integrado do Minikube. Você pode iniciá-lo executando:

```
minikube dashboard
```

Se você iniciou o processo imediatamente após executar o script de solicitação, deverá ver inicialmente uma única réplica em execução na seção `Deployments` e `Pods`:

<img src='img/initial_load.png'>

Após cerca de um minuto de execução do script, você observará que a utilização da CPU atingirá de 5 a 6m. Isso é mais do que os 20% que definimos no HPA e, portanto, acionará a ativação das réplicas adicionais:

<img src='img/autoscale_start.png'>

Finalmente, todos os três pods estarão prontos para aceitar solicitações e compartilharão a carga. Veja que cada pod abaixo mostra `2.00m` de uso da CPU.

<img src='img/autoscaled.png'>

Agora você pode interromper o script `request.sh` pressionando `Ctrl+ C`. Ao contrário do aumento de escala, a redução do número de pods levará mais tempo antes de ser executada. Você aguardará cerca de 5 minutos (quando a utilização da CPU estiver abaixo de 1m) antes de ver que há apenas um pod em execução novamente. Esse é o comportamento da versão da API `autoscaling/v1` que estamos usando. Já existe uma versão `v2` em fase beta sendo desenvolvida para substituir esse comportamento e você pode ler mais sobre ela [aqui](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#api-object).

## Tear Down

Quando terminar de experimentar, você poderá destruir os recursos que criou, se desejar. Você pode simplesmente chamar `kubectl delete -f yaml` para excluir todos os recursos definidos na pasta `yaml`. Você verá algo parecido com isto:

```
horizontalpodautoscaler.autoscaling "tf-serving-hpa" deleted
configmap "tfserving-configs" deleted
deployment.apps "tf-serving-deployment" deleted
service "tf-serving-service" deleted
```

Você pode recriar todos eles na próxima vez com um único comando executando `kubectl apply -f yaml`. Lembre-se apenas de verificar se o `metrics-server` está ativado e em execução.

Se você também quiser destruir a VM, poderá executar o comando `minikube delete`. 

Se você usou o Powershell, poderá desativar o scripting conforme mencionado nas instruções anteriores.

## Resumo

Neste laboratório, você praticou mais noções básicas do Kubernetes usando arquivos de configuração YAML e executando o contêiner Tensorflow Serving. Essa pode ser uma linha de base para você começar a tentar atender aos seus próprios modelos com uma estrutura de orquestração em mente. A maioria dos conceitos aqui será aplicável ao Qwiklabs desta semana, portanto, fique à vontade para consultar este documento. Bom trabalho e vamos para a próxima parte do curso!

