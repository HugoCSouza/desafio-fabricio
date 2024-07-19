# Desafio de desenvolvimento

## Primeiro passo - Kong

O inicio do desenvolvimento do projeto passa pela iniciação da API Gateway Kong via docker. Ela quem vai gerenciar o acesso aos microsservicos que vamos subir posteriormente

### Configuração do docker-compose

O docker compose é o arquivo que contem as instruções e configurações para que você gerencie varios conteineres de uma vez só, ao invés de ficar iniciando e configurando container por container.  

### Definição do endereço das portas

- 8000 : Acesso ao gateway de entrada
- 8001 : API Admin

### Configuração e Inicialização da imagem docker

Depois de buildar a imagem, faça primeiro a inicialização do container pelo db

### Kong

``` CLI
docker compose -f .\docker\kong\docker-compose.yml up db -d
```

Depois crie a migration para conexão do kong com o banco de dados

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong-migration
```

vai ter a resposta tal ...

Dai, tu suba o kong.

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong -d
```

Tente verificar se o kong subiu via web ou curl através do <http://localhost:8001>. Verificado que o kong está online, verifique a conexão do kong com o banco de dados, utilizando o curl.

```CLI
# Execute uma linha por vez

$response = Invoke-WebRequest -Uri "http://localhost:8001"
$json = $response.Content | ConvertFrom-Json
$json.plugins.available_on_server.oidc

```

O comando Invoke-Webrequest vai fazendo um solicitação HTTP a URI/URL informada como argumento após o argumento nomeado "-Uri" e armazena na variável *response*. Após isso, pega o conteudo da variável e converte para json e armazena na varíavel *json*. Então, se verifica se o Plugin OIDC está disponivel dentro do Kong. Se a resposta destes comandos foi "True", o serviço está ativo.

Verificado se o plugin está funcionando, crie os serviços e rotas para conectar o Kong ao Mockbin, plugin responsável por criar(mocar, criar objetos que simulem o consumo da API) endpoints para testar a API. Para criar a primeira rota:

```CLI
# Execute uma linha por vez

$uri = "http://localhost:8001/services"

$body = @{
    name = "openid-connect"
    url = "http://httpbin.org/anything"
}

$headers = @{
    "Content-Type" = "application/x-www-form-urlencoded"
}

$response = Invoke-WebRequest -Uri $uri -Method POST -Body $body -Headers $headers

$json = $response.Content | ConvertFrom-Json

$json | ConvertTo-Json -Depth 10

```

Nos variáveis body e headers, o @ é utilizado para definir uma tabela hash (chave-valor). Já o |(pipe) é utilizado para passar a saida de um comando para outro. A resposta da variável Json convertida deve ter aproximadamente este formato:

```Json
{
    "write_timeout":  60000,
    "created_at":  1720974560,
    "tls_verify":  null,
    "retries":  5,
    "path":  "/request",
    "tls_verify_depth":  null,
    "port":  80,
    "client_certificate":  null,
    "id":  "6d8b4780-f609-457d-acea-afe814f34279",
    "protocol":  "http",
    "ca_certificates":  null,
    "connect_timeout":  60000,
    "enabled":  true,
    "read_timeout":  60000,
    "name":  "mock-service",
    "tags":  null,
    "host":  "mockbin.org",
    "updated_at":  1720974560
}
```

Com essa resposta, usaremos para criar uma rota, através do ID.

```CLI
# Execute uma linha por vez

$uri = "http://localhost:8001/services/openid-connect/routes"

$body = @{
    "name" = "openid-connect"
    "paths[]" = "/"
}

$response = Invoke-WebRequest -Uri $uri -Method POST -Body $body

$json = $response.Content | ConvertFrom-Json

$json | ConvertTo-Json -Depth 10

```

E terá como resposta, um arquivo Json

```JSON
{
    "https_redirect_status_code":  426,
    "hosts":  null,
    "created_at":  1720982646,
    "protocols":  [
                      "http",
                      "https"
                  ],
    "headers":  null,
    "snis":  null,
    "id":  "a5b3d33b-b07b-4ea7-a283-1b3f0c05c5a4",
    "regex_priority":  0,
    "tags":  null,
    "service":  {
                    "id":  "6d8b4780-f609-457d-acea-afe814f34279"
                },
    "response_buffering":  true,
    "request_buffering":  true,
    "strip_path":  true,
    "destinations":  null,
    "updated_at":  1720982646,
    "path_handling":  "v0",
    "preserve_host":  false,
    "paths":  [
                  "/mock"
              ],
    "methods":  null,
    "sources":  null,
    "name":  null
}
```

Verificando, se tudo ocorreu corretamente, faça um cURL na rota:

```CLI
# Execute uma linha por vez

$uri = "http://localhost:8000/mock"

$response = Invoke-WebRequest  -Uri $uri -Method Get     

$responseJson = $response |ConvertTo-Json -Depth 10

Write-Output $formattedResponse

```

### Keycloak

Configurado seu docker-compose corretamente, suba primeiro o banco de dados do keycloak

```CLI
docker compose -f .\docker\kong\docker-compose.yml up keycloak-db -d
```

Após verificar que tudo está correto e que o banco de dados subiu corretamente no container, suba o keycloak.

```CLI
docker compose -f .\docker\kong\docker-compose.yml up keycloak -d
```

Após isso faz as confirações do site lá;

### OIDC

Para configurar o OIDC (plugin de autenticação), é preciso fornecer a ele, o Client ID, o Cliente Secret e o endpoint discovery.  Para acessar o discovery endpoint, que é onde o kong irá pegar informaçãoes para a autenticação, execute as linhas abaixo.

```CLI
# Execute uma linha por vez

$uri = "http://localhost:8180/realms/master/.well-known/openid-configuration"

$response = Invoke-RestMethod -Uri $uri -Method Get

$formattedResponse = $response | ConvertTo-Json -Depth 10

Write-Output $formattedResponse

```

Por estar "Conteinerizado", cada imagem está no seu proprio container, portanto, o localhost do kong só é acessivel a ele mesmo. Para permiti-lo acessar o keycloack para fazer a verificação de ID, encontre o ip da sua maquina, o secret do keycloak para endereçar para o kong

```CLI
$host_ip = "192.168.100.15" #Descubra seu ip com ipconfig

$secret_client = "mKcDAbwyQy1rorkka4ZgUvLqSat9UgAm" #Está no keycloak no client criado "kong"

$uri = "http://localhost:8001/plugins"

$body = @{
    name = "oidc"
    config = @{
        client_id = "kong"
        client_secret = ${secret_client}
        discovery = "http://${host_ip}:8180/realms/master/.well-known/openid-configuration"
    }
}
$jsonBody = $body | ConvertTo-Json

$response = Invoke-WebRequest -Uri $uri -Method POST -Body $jsonBody -ContentType "application/json"

$json = $response.Content | ConvertFrom-Json

$formattedResponse = $json | ConvertTo-Json -Depth 10

Write-Output $formattedResponse

```

### Insomnia

Como ao tentar mockar via mockbin não foi possível, é apresentado no site uma alternativa chamada Insomnia. É um app para realizar mockagens para endpoins e fazer requisições HTTP para testar APIs (Estilo PostMan). Baixe o [Insomnia](https://insomnia.rest/download) no site oficial e instale no seu SO.
