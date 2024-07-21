# Desafio de desenvolvimento

## Primeiro passo - Kong

O inicio do desenvolvimento do projeto passa pela iniciação da API Gateway Kong via docker. Ela quem vai gerenciar o acesso aos microsservicos que vamos subir posteriormente

### Configuração do docker-compose

O docker compose é o arquivo que contem as instruções e configurações para que você gerencie varios conteineres de uma vez só, ao invés de ficar iniciando e configurando container por container.  

### Definição do endereço das portas

- 8000 : Acesso ao gateway de entrada
- 8001 : API Admin

### Configuração e Inicialização da imagem docker

Depois de buildar a imagem, faça primeiro a inicialização do container pelos bancos de dados, tanto do kong quanto do keycloack

``` CLI
docker compose -f .\docker\kong\docker-compose.yml up db keycloak-db -d
```

### Kong

Depois crie a migration para conexão do kong com o banco de dados

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong-migration -d
```

A migration irá iniciar e finalizar, é o normal. Ela cria os caminhos de conexão entre o Kong e o banco de dados. Dai, tu suba o kong e o Keycloak.

```CLI
docker compose -f .\docker\kong\docker-compose.yml up kong keycloak -d
```

Tente verificar se o kong subiu via web ou curl através do <http://localhost:8001>. Verificado que o kong está online, verifique a conexão do kong com o banco de dados, utilizando o curl.

```CLI
# Execute uma linha por vez

$response = Invoke-WebRequest -Uri "http://localhost:8001"
$json = $response.Content | ConvertFrom-Json
$json.plugins.available_on_server.oidc

```

O comando Invoke-Webrequest vai fazendo um solicitação HTTP a URI/URL informada como argumento após o argumento nomeado "-Uri" e armazena na variável *response*. Após isso, pega o conteudo da variável e converte para json e armazena na varíavel *json*. Então, se verifica se o Plugin OIDC está disponivel dentro do Kong. Se a resposta destes comandos foi "True", o serviço está ativo.

Verificado se o plugin está funcionando, crie os serviços e rotas para conectar o Kong ao gateway.

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
    "port":  80,
    "updated_at":  1721522138,
    "client_certificate":  null,
    "protocol":  "http",
    "name":  "openid-connect",
    "connect_timeout":  60000,
    "read_timeout":  60000,
    "path":  "/anything",
    "enabled":  true,
    "host":  "httpbin.org",
    "id":  "5f23bbf2-a215-4bdc-9d26-d022c4e00a60",
    "tls_verify_depth":  null,
    "created_at":  1721522138,
    "retries":  5,
    "write_timeout":  60000,
    "tags":  null,
    "ca_certificates":  null,
    "tls_verify":  null
}
```

Com essa resposta, usaremos para criar uma rota, através do para o serviço de conexão.

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
    "created_at":  1721522166,
    "updated_at":  1721522166,
    "name":  "openid-connect",
    "service":  {
                    "id":  "5f23bbf2-a215-4bdc-9d26-d022c4e00a60"
                },
    "path_handling":  "v0",
    "regex_priority":  0,
    "https_redirect_status_code":  426,
    "headers":  null,
    "request_buffering":  true,
    "response_buffering":  true,
    "id":  "09e28a69-f115-4470-95b7-4dc81d425add",
    "methods":  null,
    "hosts":  null,
    "preserve_host":  false,
    "destinations":  null,
    "snis":  null,
    "strip_path":  true,
    "sources":  null,
    "tags":  null,
    "paths":  [
                  "/"
              ],
    "protocols":  [
                      "http",
                      "https"
                  ]
}
```

Após isso, deve-se configurar a sessão de autenticação do Kong via Keycloak. Para isso será necessário o ID do client service.

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

$secret_client = "wezXLgiseeeKCz0W6KnnKQoteVt1wbT7" #Está no keycloak no client criado "kong"

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

### Conexão

```CLI
# Execute uma linha por vez

# 
$user = "user"
$password = "123"
$url = "http://localhost:8000"
$cookieFile = "example-user"

# Crie uma instância de WebRequest com autenticação e cookies
Invoke-WebRequest -Uri $url -Credential (New-Object PSCredential($user, (ConvertTo-SecureString $password -AsPlainText -Force))) -SessionVariable webSession

# Salve os cookies da requisição
$cookies = $webSession.Cookies.GetCookies($url)

$cookies | ForEach-Object {
    "$($_.Name)=$($_.Value)"
} | Out-File -FilePath $cookieFile


```

### Insomnia

Como ao tentar mockar via mockbin não foi possível, é apresentado no site uma alternativa chamada Insomnia. É um app para realizar mockagens para endpoins e fazer requisições HTTP para testar APIs (Estilo PostMan). Baixe o [Insomnia](https://insomnia.rest/download) no site oficial e instale no seu SO.
