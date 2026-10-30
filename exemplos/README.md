# Laboratório — Traefik Proxy Reverso com Docker

## 1. Objetivo

Este laboratório demonstra, de maneira prática, a utilização do Traefik como proxy reverso para encaminhar requisições HTTP a uma aplicação executada em um contêiner Docker.

A estrutura utiliza:

- Traefik como proxy reverso.
- Docker Compose para gerenciamento dos serviços.
- Aplicação Whoami como destino de demonstração.
- Rede Docker privada para comunicação entre os contêineres.
- Roteamento baseado no cabeçalho HTTP `Host`.

O ambiente foi desenvolvido para execução local e não deve ser considerado uma configuração pronta para produção.

## 2. Estrutura dos arquivos

```text
exemplos/
│
├── README.md
├── docker-compose.yml
├── traefik.yml
│
└── rotas/
    └── exemplo-app.yml
```

## 3. Pré-requisitos

Antes de executar o laboratório, tenha instalado:

- Docker Engine.
- Docker Compose v2.
- Acesso ao terminal.
- Conectividade com o registry de imagens Docker para baixar as imagens necessárias.

Verifique as versões:

```bash
docker --version
docker compose version
```

## 4. Validar a configuração

Na raiz do repositório, execute:

```bash
docker compose -f exemplos/docker-compose.yml config -q
```

O comando verifica a configuração do Compose sem iniciar os contêineres.

Se não houver erros, prossiga.

## 5. Iniciar o laboratório

Execute:

```bash
docker compose -f exemplos/docker-compose.yml up -d
```

O Docker irá:

1. Baixar as imagens necessárias, caso ainda não estejam disponíveis.
2. Criar a rede Docker do laboratório.
3. Criar o contêiner Traefik.
4. Criar a aplicação Whoami.
5. Iniciar os serviços.

## 6. Verificar os contêineres

Execute:

```bash
docker compose -f exemplos/docker-compose.yml ps
```

Os serviços `traefik-demo` e `web-app-demo` devem aparecer em execução.

Para consultar os logs do Traefik:

```bash
docker compose -f exemplos/docker-compose.yml logs --tail 50 traefik
```

Para acompanhar os logs em tempo real:

```bash
docker compose -f exemplos/docker-compose.yml logs -f traefik
```

## 7. Testar o roteamento

O laboratório publica a porta 8080 do host somente no endereço de loopback e a encaminha para a porta 80 do contêiner Traefik (`127.0.0.1:8080:80`). O Traefik escuta na porta 80 do contêiner. A rota encaminha para a porta interna da aplicação; neste laboratório, o Whoami escuta na porta 80 (`http://web-app:80`). Em outras aplicações, confirme a porta interna em que o serviço escuta.

Para testar a regra de roteamento sem precisar criar um registro DNS, execute:

```bash
curl -i \
  -H 'Host: app.example.internal' \
  http://127.0.0.1:8080/
```

O cabeçalho `Host` simula uma requisição destinada ao domínio configurado no arquivo dinâmico.

A aplicação Whoami deverá retornar informações sobre a requisição recebida, incluindo dados do ambiente que a processou.

Esse resultado demonstra que:

- A conexão chegou ao Traefik.
- A regra de domínio foi identificada.
- O proxy encaminhou a requisição.
- A aplicação de destino respondeu.

## 8. Testar no navegador

Também é possível testar utilizando o navegador.

Como a publicação está limitada a `127.0.0.1`, o acesso deverá ser realizado no próprio computador em que o laboratório está executando.

Para simular o domínio, seria necessário configurar uma resolução local para `app.example.internal` apontando para `127.0.0.1`, por exemplo, no arquivo hosts do sistema operacional.

Alternativamente, o teste com `curl` apresentado anteriormente já valida o roteamento sem necessidade dessa alteração.

## 9. Verificar a rede Docker

Para inspecionar a rede criada:

```bash
docker network inspect proxy-net
```

Os contêineres Traefik e Whoami devem estar conectados à mesma rede.

A aplicação não possui uma porta publicada diretamente no host. Seu acesso ocorre por meio do proxy; a porta interna 80 é específica deste exemplo Whoami.

## 10. Encerrar o laboratório

Para parar e remover os contêineres e a rede criada pelo Compose:

```bash
docker compose -f exemplos/docker-compose.yml down
```

Esse comando não remove as imagens baixadas.

## 11. Diagnóstico rápido

### Traefik não inicia

```bash
docker compose -f exemplos/docker-compose.yml logs traefik
```

Verifique a configuração estática, o arquivo dinâmico e a publicação da porta.

### Aplicação não responde

```bash
docker compose -f exemplos/docker-compose.yml ps
docker network inspect proxy-net
```

Confirme se os dois contêineres estão ativos e conectados à rede.

### Erro 404 do Traefik

Verifique se o cabeçalho `Host` enviado na requisição corresponde exatamente à regra configurada em `exemplo-app.yml`.

### Porta 8080 ocupada

Verifique se outro serviço já está utilizando a porta local:

```bash
sudo ss -lntp | grep ':8080'
```

Se necessário, escolha outra porta livre e ajuste o mapeamento no Docker Compose.

## 12. Considerações de segurança

Este laboratório utiliza HTTP e foi projetado para testes locais.

A publicação em `127.0.0.1:8080` evita disponibilizar o serviço diretamente nas interfaces de rede do computador.

Em ambientes corporativos ou de produção, é necessário planejar:

- Restrições de origem no firewall.
- Publicação controlada das portas.
- HTTPS com certificado confiável.
- Proteção das chaves privadas.
- Monitoramento e atualização dos contêineres.
- Revisão das permissões e da arquitetura de rede.

Não utilize este laboratório como substituto de uma implantação de produção sem revisar os requisitos de segurança.

## 13. Resultado esperado

Ao finalizar o laboratório, será possível observar o Traefik recebendo uma requisição HTTP e encaminhando-a para a aplicação Whoami conforme o domínio informado no cabeçalho `Host`.

O exercício demonstra o funcionamento básico do proxy reverso e serve como ponto de partida para publicação de múltiplas aplicações.

---

*Laboratório demonstrativo. Nomes, domínios e configurações são fictícios e não representam informações operacionais de uma infraestrutura corporativa.*
