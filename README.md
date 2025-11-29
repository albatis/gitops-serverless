# GitOps Serverless

Este repositório contém os artefatos para a implantação, seguindo a metodologia GitOps, de um ambiente serverless (ou o mais próximo possível usando Kubernetes). A arquitetura utiliza um runtime que executa uma função serverless em Python, lendo métricas de uma fila Redis e publicando o resultado processado em outra.

## Estrutura do Projeto e Artefatos


O projeto é composto por dois manifestos Kubernetes principais:

- **configmap.yaml**: Define dois ConfigMaps.

    - O primeiro ConfigMap, outputkey, armazena a chave de saída do Redis (alexandrevieira-proj3-output) usada pelo runtime serverless para publicar os resultados das métricas processadas.

    - O segundo ConfigMap, pyfile, armazena o código Python da função serverless em um arquivo chamado pyfile. Este código contém a função handler que processa as métricas.

- **serverless-deployment-course.yaml**: Contém o manifesto Deployment do Kubernetes que implementa o runtime serverless (serverless-redis).

    - O Deployment monta o código Python (do ConfigMap pyfile) como um arquivo dentro do contêiner (/opt/usermodule.py).

    - Configura variáveis de ambiente (como REDIS_HOST, REDIS_PORT, REDIS_INPUT_KEY) para conectar-se ao Redis.

    - A chave de saída do Redis (REDIS_OUTPUT_KEY) é injetada dinamicamente através do ConfigMap outputkey, demonstrando uma prática de configuração desacoplada.

## Descrição da Função Serverless (pyfile no configmap.yaml)

A função Python handler(input: dict, context: object) é o coração da lógica de processamento de métricas.

1. Processamento de Tráfego de Rede (Egresso):

    - Calcula a porcentagem de tráfego de rede de saída (egresso) (percent-network-egress) com base nos bytes enviados (bytes_sent) em relação ao total de bytes (enviados + recebidos) da interface eth0.

2. Processamento de Caching de Memória:

    - Calcula a porcentagem de memória usada para buffers e cache (percent-memory-caching) em relação à memória total do sistema.

3. Cálculo de Utilização Média da CPU (Janela Temporal):

    - Mantém um histórico (context.env) da utilização de cada núcleo de CPU.

    - Utiliza uma janela deslizante (sliding window) de tamanho WINDOW_SIZE = 12 para armazenar as últimas 12 amostras de utilização de cada CPU.

    - Calcula a utilização média (avg-util-cpuX-60sec) de cada núcleo de CPU ao longo dessa janela de tempo, que é de aproximadamente 60 segundos (assumindo que as amostras chegam a cada 5 segundos).

## Fluxo de Implantação e Operação

1. Implantação GitOps: O ArgoCD (ou ferramenta GitOps similar) monitora este repositório. Ao detectar os manifestos, ele garante que os ConfigMaps e o Deployment sejam aplicados no cluster Kubernetes.

2. Injeção de Código e Configuração: O Deployment do serverless-redis é criado. O código Python e as chaves do Redis são injetadas no contêiner a partir dos ConfigMaps.

3. Execução Serverless: O runtime dentro do contêiner:

    - Lê dados de métricas (input) da chave Redis metrics.

    - Executa a função handler para processar essas métricas.

    - Grava os resultados processados (output) na chave Redis alexandrevieira-proj3-output.

    - O contexto (context.env) é usado para manter o estado entre as invocações, especificamente para o cálculo da média móvel de utilização da CPU.