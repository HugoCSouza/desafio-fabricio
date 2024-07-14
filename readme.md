# Desafio de desenvolvimento

## Primeiro passo - Kong

O inicio do desenvolvimento do projeto passa pela iniciação da API Gateway Kong via docker. Ela quem vai gerenciar o acesso aos microsservicos que vamos subir posteriormente

### Configuração do docker-compose

O docker compose é o arquivo que contem as instruções e configurações para que você gerencie varios conteineres de uma vez só, ao invés de ficar iniciando e configurando container por container.  

### Definição do endereço das portas

- 8000 : Acesso ao gateway de entrada
- 8001 : API Admin

Depois de buildar a imagem, faça primeiro a inicialização do container pelo db

``` CLI
docker compose -f .\docker\kong\docker-compose.yml up db
```

Depois crie a migration para conexão do kong com o banco de dados

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong-migrations
```

vai ter a resposta tal ... 

Dai, tu suba o kong.

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong
```

Tente verificar se o kong subiu via web ou curl através do <http://localhost:8001>. Verificado que o kong está online, verifique a conexão do kong com o banco de dados, utilizando o curl.

```CLI
# Execute uma linha por vez

$response = Invoke-WebRequest -Uri "http://localhost:8001"
$json = $response.Content | ConvertFrom-Json
$json.plugins.available_on_server.oidc

```

O comando Invoke-Webrequest vai fazendo um solicitação HTTP a URI/URL informada como argumento após o argumento nomeado "-Uri" e armazena na variável *response*. Após isso, pega o conteudo da variável e converte para json e armazena na varíavel *json*. Então, se verifica se o Plugin OIDC está disponivel dentro do Kong. Se a resposta destes comandos foi "True", o serviço está ativo.

Verificado se o plugin está funcionando, crie os serviços e rotas para conectar o Kong ao Mockbin, plugin responsável por criar(mockar) endpoints para testar a API. Para criar a primeira rota:

```CLI
# Execute uma linha por vez

$uri = "http://localhost:8001/services"

$body = @{
    name = "mock-service"
    url = "http://mockbin.org/request"
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

$uri = "http://localhost:8001/routes"

$body = @{
    "service.id" = "6d8b4780-f609-457d-acea-afe814f34279"
    "paths[]" = "/mock"
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

$response = Invoke-RestMethod  -Uri $uri -Method Get     

$responseJson = $response |ConvertTo-Json -Depth 10

Write-Output $formattedResponse

```