# Implementação — Traefik Proxy Reverso com Docker

## 1. Introdução

Este documento apresenta a sequência técnica de implementação do Traefik como proxy reverso para publicação centralizada de aplicações web em um ambiente Linux com Docker.

A implantação foi organizada em etapas para facilitar a manutenção, permitir a expansão da arquitetura e reduzir riscos durante as alterações na infraestrutura.

Os exemplos utilizam endereços, nomes de domínio e caminhos fictícios. Eles devem ser adaptados antes da utilização em qualquer ambiente.

## 2. Pré-requisitos

Para reproduzir a implementação, considere os seguintes componentes:

| Componente | Requisito |
|---|---|
| Sistema operacional | Linux Server |
| Contêineres | Docker Engine |
| Orquestração | Docker Compose |
| Proxy reverso | Traefik v3 |
| Rede | Rede Docker compartilhada |
| DNS | Registro interno para a aplicação |
| Firewall | iptables e/ou UFW |
| Aplicação | Serviço web ativo em porta interna conhecida |

Também é necessário possuir privilégios administrativos para configurar o Docker, o firewall e os arquivos do sistema.

## 3. Preparação do ambiente

Antes de iniciar, verifique a instalação do Docker e do Docker Compose.

### Verificar Docker Engine

```bash
docker --version
```

### Verificar Docker Compose

```bash
docker compose version
```

### Verificar os contêineres existentes

```bash
sudo docker ps
```

### Verificar as redes Docker

```bash
sudo docker network ls
```

A rede que será utilizada pelo proxy precisa permitir comunicação com as aplicações de destino.

A versão usada no exemplo do laboratório é fixada no Compose (`traefik:v3.3.6`). Em ambientes reais, valide a versão adotada e sua política de atualização antes de aplicar qualquer mudança.

**Importante:** antes de modificar um ambiente existente, faça backup dos arquivos de configuração e identifique os serviços que não devem ser interrompidos.

## 4. Estrutura de diretórios

Uma organização possível para os arquivos do Traefik é:

```text
/etc/traefik/
│
├── traefik.yml
│
└── dynamic/
    ├── web-app.yml
    ├── sistema-a.yml
    └── sistema-b.yml
```

O arquivo `traefik.yml` concentra as configurações estáticas do proxy.

O diretório `dynamic` contém as definições de roteamento e os serviços de destino.

### Criar o diretório de configurações

```bash
sudo install -d -o root -g root -m 750 /etc/traefik/dynamic
```

As permissões devem impedir que usuários não autorizados alterem as configurações do proxy.

## 5. Configuração estática do Traefik

Crie o arquivo:

```bash
sudo nano /etc/traefik/traefik.yml
```

Exemplo de configuração HTTP inicial:

```yaml
api:
  dashboard: false

entryPoints:
  web:
    address: ":80"

providers:
  file:
    directory: /etc/traefik/dynamic
    watch: true

log:
  level: INFO

accessLog: {}
```

### Descrição dos parâmetros

| Parâmetro | Função |
|---|---|
| `api.dashboard` | Desativa o dashboard administrativo |
| `entryPoints` | Define os pontos de entrada |
| `web` | Identifica o ponto de entrada HTTP |
| `address` | Define a porta de escuta dentro do contêiner |
| `providers.file` | Habilita a leitura de arquivos de configuração |
| `directory` | Diretório das rotas dinâmicas |
| `watch` | Monitora alterações nos arquivos |
| `log.level` | Define o nível dos logs |
| `accessLog` | Habilita os registros de acesso |

O endereço `:80` faz o Traefik escutar na porta 80 dentro do contêiner.

O IP do host deve ser configurado separadamente no mapeamento de portas do Docker.

## 6. Configuração dinâmica de uma aplicação

Crie um arquivo para a primeira aplicação:

```bash
sudo nano /etc/traefik/dynamic/web-app.yml
```

Exemplo:

```yaml
http:
  routers:
    web-app:
      rule: "Host(`app.example.internal`)"
      entryPoints:
        - web
      service: web-app

  services:
    web-app:
      loadBalancer:
        servers:
          - url: "http://web-app:80"
```

### Entendendo a configuração

**Router**

Identifica as requisições destinadas ao domínio definido na regra `Host`.

**EntryPoint**

Indica por qual ponto de entrada a requisição será recebida.

**Service**

Define o serviço de destino.

**Load Balancer**

Encaminha a requisição para o endereço interno da aplicação.

O nome `web-app` utilizado no destino deve ser resolvido pela rede Docker compartilhada. A porta `80` é específica do exemplo Whoami; para outra aplicação, informe a porta em que ela escuta dentro da rede Docker. Essa porta é independente da porta publicada no host.

## 7. Configuração do Docker Compose

O Traefik foi adicionado como um serviço independente ao arquivo Compose do ambiente.

Exemplo simplificado:

```yaml
services:
  traefik:
    image: traefik:v3.3.6
    container_name: traefik
    restart: unless-stopped
    init: true
    mem_limit: 512m
    cpus: 0.5
    pids_limit: 256

    ports:
      - "192.0.2.10:80:80"

    volumes:
      - /etc/traefik/traefik.yml:/etc/traefik/traefik.yml:ro
      - /etc/traefik/dynamic:/etc/traefik/dynamic:ro

    networks:
      - proxy-net

    logging:
      driver: json-file
      options:
        max-size: 10m
        max-file: "5"

networks:
  proxy-net:
    external: true
```

O exemplo deve ser adaptado à estrutura de redes e serviços do ambiente.

### Decisões de configuração

- `restart: unless-stopped`: permite reinicialização automática do contêiner.
- `mem_limit`: estabelece limite de memória.
- `cpus`: limita o consumo de CPU.
- `pids_limit`: limita a quantidade de processos.
- `ports`: publica a porta HTTP no endereço específico do host.
- `:ro`: monta os arquivos de configuração como somente leitura.
- `networks`: conecta o proxy à rede Docker apropriada.
- `logging`: configura rotação dos logs.

O socket Docker não é montado no contêiner, pois a configuração de rotas é realizada pelo provedor de arquivos.

## 8. Validação do Docker Compose

Antes de iniciar o serviço, valide a sintaxe do arquivo.

```bash
sudo docker compose -f /path/to/compose.yaml config -q
```

A validação deve ser concluída sem erros.

Verifique também se os arquivos de configuração montados no contêiner existem no host.

## 9. Implantação do Traefik

Com as configurações validadas, inicie somente o serviço do proxy:

```bash
sudo docker compose -f /path/to/compose.yaml up -d traefik
```

Essa abordagem evita iniciar ou recriar desnecessariamente os demais serviços declarados no Compose.

### Verificar o estado

```bash
sudo docker ps --filter name=traefik
```

### Consultar os logs

```bash
sudo docker logs --tail 80 traefik
```

O contêiner deve permanecer em execução sem erros de inicialização.

## 10. Integração com DNS

Após identificar o endereço IP utilizado para publicar o proxy, crie um registro DNS do tipo A.

Exemplo:

| Tipo | Nome | Endereço |
|---|---|---|
| A | app.example.internal | 192.0.2.10 |

A criação do registro deve seguir a política de administração DNS do ambiente.

### Validar a resolução

Em um cliente Windows:

```cmd
nslookup app.example.internal
```

A resposta deve apresentar o endereço IP esperado.

## 11. Configuração do firewall

A publicação de portas pelo Docker exige atenção ao fluxo de encaminhamento de pacotes.

Para restringir o acesso às portas publicadas, deve-se avaliar a utilização da cadeia `DOCKER-USER`.

A política de acesso deve contemplar:

- Permissão para as origens autorizadas.
- Bloqueio para origens não autorizadas.
- Preservação das conexões estabelecidas.
- Preservação das regras de acesso administrativo existentes.
- Persistência das regras após reinicializações.

Os comandos de inspeção incluem:

```bash
sudo iptables -S DOCKER-USER
sudo iptables -vnL DOCKER-USER
sudo ufw status verbose
```

As regras específicas devem ser mantidas em um procedimento controlado, com backup e plano de rollback.

Consulte também o documento [Segurança e Firewall](seguranca-e-firewall.md).

## 12. Testes de roteamento

Antes de depender exclusivamente do DNS, é possível testar a regra utilizando diretamente o endereço IP do servidor e informando o cabeçalho `Host`.

```bash
curl -sS -D - -o /dev/null --max-time 5 \
  -H 'Host: app.example.internal' \
  http://192.0.2.10/
```

Esse teste permite verificar se o Traefik identifica o domínio e encaminha a requisição ao serviço correto.

Uma resposta HTTP gerada pela aplicação, como um redirecionamento para uma página de login, indica que o encaminhamento foi realizado.

## 13. Teste pelo navegador

Depois de validar o DNS e o roteamento, acesse:

```text
http://app.example.internal
```

Verifique:

- Se o domínio resolve corretamente.
- Se a aplicação é carregada.
- Se o redirecionamento esperado ocorre.
- Se a página de autenticação está disponível.
- Se os logs do Traefik registram a requisição.

A validação pelo navegador complementa os testes de rede e de linha de comando.

## 14. Diagnóstico de problemas

### Erro: cannot assign requested address

Mensagem semelhante a:

```text
listen tcp 192.0.2.10:80: bind: cannot assign requested address
```

Uma causa possível é configurar o Traefik para escutar no endereço IP do host dentro do contêiner.

Correção:

```yaml
entryPoints:
  web:
    address: ":80"
```

O endereço IP do host deve permanecer no mapeamento de portas do Docker Compose.

### Erro de conexão recusada

Verifique:

```bash
sudo docker ps
sudo docker logs --tail 80 traefik
sudo docker port traefik
```

Também verifique o firewall, a publicação da porta e o estado da aplicação de destino.

### DNS não resolve

Verifique:

```cmd
nslookup app.example.internal
```

Confirme a existência do registro, a zona consultada e o servidor DNS utilizado pelo cliente.

### Aplicação não responde pelo proxy

Verifique:

- Se a aplicação está em execução.
- Se a porta interna está correta.
- Se o proxy e a aplicação compartilham uma rede Docker apropriada.
- Se o nome do serviço é resolvível pelo contêiner Traefik.
- Se o arquivo de rota está sintaticamente correto.

## 15. Expansão para outras aplicações

Para adicionar uma nova aplicação, o processo geral consiste em:

1. Confirmar o serviço e sua porta interna.
2. Garantir a conectividade pela rede Docker.
3. Criar um arquivo dinâmico próprio.
4. Definir o domínio DNS.
5. Configurar a regra de roteamento.
6. Aplicar as regras de firewall necessárias.
7. Testar a rota antes de disponibilizar aos usuários.
8. Documentar a nova publicação.

Não é necessário publicar diretamente no host a porta web de cada aplicação quando o acesso pode ocorrer pela rede interna do Docker através do Traefik.

## 16. Considerações sobre HTTPS

A implementação inicial descrita neste estudo foi validada por HTTP.

Para utilização definitiva, é necessário configurar HTTPS com certificado apropriado, incluindo:

- Certificado válido para o domínio.
- Chave privada protegida.
- Cadeia de confiança reconhecida pelos clientes.
- EntryPoint HTTPS.
- Roteadores com TLS.
- Redirecionamento HTTP para HTTPS.
- Regras de firewall correspondentes.
- Processo de renovação do certificado.

Até essa implementação, HTTP deve ser tratado como temporário e restrito às origens autorizadas.

## 17. Conclusão

A implantação do Traefik permitiu centralizar o acesso à aplicação web, simplificar a publicação por DNS e estabelecer uma base para a expansão de outros serviços.

A sequência de implementação demonstra a importância de validar separadamente as configurações do proxy, o Docker, o DNS, o firewall e a aplicação.

A solução deve evoluir com HTTPS e revisões periódicas dos controles de segurança.

**O objetivo não é apenas fazer a aplicação funcionar, mas garantir que ela seja publicada de maneira controlada, documentada e sustentável.**

---

*Documento técnico de estudo de caso. Endereços IP, nomes DNS, caminhos e demais informações foram generalizados para preservar a confidencialidade do ambiente original.*
