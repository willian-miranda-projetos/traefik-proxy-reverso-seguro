# Segurança e Firewall — Traefik Proxy Reverso

## 1. Introdução

A segurança da conectividade foi definida como um dos requisitos fundamentais da implantação do proxy reverso.

O objetivo não era simplesmente disponibilizar uma aplicação web, mas garantir que somente as origens autorizadas pudessem estabelecer conexões com os serviços publicados no servidor.

Em ambientes que utilizam Docker, esse controle exige atenção especial ao funcionamento do firewall, principalmente quando são utilizadas portas publicadas pelos contêineres.

Este documento apresenta as estratégias de proteção adotadas, os mecanismos utilizados e os cuidados necessários para manter a exposição dos serviços sob controle.

## 2. Estratégia de segurança

A abordagem foi baseada no princípio do menor privilégio: permitir somente as conexões necessárias, a partir das origens autorizadas.

Os principais controles considerados foram:

- Restrição de acesso por endereço IP e sub-rede.
- Controle de conexões destinadas às portas publicadas pelo Docker.
- Utilização da cadeia `DOCKER-USER` do iptables.
- Preservação das regras de acesso SSH.
- Desativação do dashboard administrativo do Traefik.
- Ausência de acesso direto ao socket Docker pelo proxy.
- Montagem dos arquivos de configuração como somente leitura.
- Limitação de recursos computacionais do contêiner.
- Preparação para adoção de HTTPS.

## 3. O desafio do firewall com Docker

Uma particularidade importante do Docker é que o tráfego destinado às portas publicadas pelos contêineres pode passar por regras de encaminhamento e NAT.

Por isso, não é seguro presumir que uma regra convencional de entrada do UFW, isoladamente, controlará todo o tráfego destinado a uma porta publicada pelo Docker.

Nesse cenário, a cadeia `DOCKER-USER` oferece um ponto apropriado para aplicar controles adicionais ao tráfego encaminhado para os contêineres.

A cadeia permite avaliar as conexões antes que elas sejam processadas pelas regras específicas de encaminhamento criadas pelo Docker.

**Importante:** o comportamento efetivo depende da configuração de rede, backend de firewall e versão do Docker. Sempre valide o caminho real do tráfego no ambiente em que as regras serão aplicadas.

## 4. Controle de acesso pela cadeia DOCKER-USER

Foi utilizada a cadeia `DOCKER-USER` para controlar o acesso às portas publicadas.

A estratégia contempla três comportamentos:

1. Permitir o tráfego de conexões já estabelecidas ou relacionadas.
2. Autorizar novas conexões provenientes das redes previamente aprovadas.
3. Bloquear novas conexões de outras origens para as portas protegidas.

### Exemplo conceitual de regras

O exemplo abaixo utiliza endereços e redes fictícios. Não representa a lista de permissões do ambiente original.

```bash
# Permitir conexões estabelecidas ou relacionadas
iptables -A DOCKER-USER \
  -m conntrack --ctstate ESTABLISHED,RELATED \
  -j ACCEPT

# Permitir HTTP de uma rede autorizada
iptables -A DOCKER-USER \
  -p tcp \
  -s 10.20.0.0/16 \
  -m conntrack --ctstate NEW \
  --ctorigdst 10.20.30.40 \
  --ctorigdstport 80 \
  -j ACCEPT

# Bloquear HTTP para as demais origens
iptables -A DOCKER-USER \
  -p tcp \
  -m conntrack \
  --ctorigdst 10.20.30.40 \
  --ctorigdstport 80 \
  -j DROP
```

**Atenção:** esse trecho é ilustrativo, não um script pronto para produção. Em uma implantação real, as regras devem ser adaptadas às redes autorizadas, ao endereço original de destino, à ordem das regras existentes e à política de acesso.

O uso de `--ctorigdst` e `--ctorigdstport` permite avaliar o endereço e a porta de destino originais da conexão, mesmo após o processamento de DNAT pelo Docker.

## 5. Preservação do acesso SSH

Além da publicação web, o servidor possui serviços administrativos acessíveis por portas SSH publicadas pelo Docker.

Essas conexões possuem uma política de acesso própria, com origens explicitamente autorizadas.

A implementação do controle HTTP foi realizada preservando as regras SSH existentes.

Essa separação é importante porque a publicação de uma aplicação web não deve modificar ou ampliar inadvertidamente o acesso administrativo ao servidor.

As portas e endereços utilizados na implantação original foram omitidos desta documentação por motivos de confidencialidade.

## 6. Persistência das regras

As regras de firewall precisam continuar aplicadas após reinicializações do servidor ou do Docker.

Para isso, foi utilizado um script administrativo executado por uma unidade do systemd.

A implementação utiliza verificações com `iptables -C` antes de inserir regras, evitando duplicações em execuções sucessivas.

Exemplo conceitual:

```bash
#!/bin/sh
set -eu

IPT=/usr/sbin/iptables
CHAIN=DOCKER-USER

add_rule() {
    "$IPT" -C "$CHAIN" "$@" 2>/dev/null || \
        "$IPT" -A "$CHAIN" "$@"
}
```

O restante do script deve conter as regras específicas de autorização e bloqueio definidas para o ambiente.

A unidade systemd permite executar o script de forma controlada durante a inicialização.

É importante verificar a ordem de inicialização e garantir que a cadeia `DOCKER-USER` exista quando o script for executado.

## 7. Inspeção e validação do firewall

A validação foi realizada por meio da inspeção das regras efetivamente carregadas no sistema.

Comandos utilizados:

### Listar as regras da cadeia

```bash
sudo iptables -S DOCKER-USER
```

### Consultar regras e contadores de pacotes

```bash
sudo iptables -vnL DOCKER-USER
```

### Consultar o estado do UFW

```bash
sudo ufw status verbose
```

### Verificar o serviço persistente

```bash
sudo systemctl status proxy-docker-firewall.service --no-pager
```

### Consultar o script de firewall

```bash
sudo cat /usr/local/sbin/proxy-docker-firewall
```

A inspeção permite confirmar a presença das regras, sua ordem e os contadores de tráfego.

## 8. Testes de segurança recomendados

A validação de firewall não deve se limitar à existência das regras.

É necessário testar o comportamento real a partir de diferentes origens.

| Cenário | Resultado esperado |
|---|---|
| Cliente de rede autorizada | Conexão permitida |
| Cliente fora das redes autorizadas | Conexão bloqueada |
| Conexão já estabelecida | Tráfego permitido conforme o estado da conexão |
| Acesso SSH de origem autorizada | Acesso preservado |
| Acesso SSH de origem não autorizada | Conexão bloqueada |
| Acesso à aplicação pelo domínio DNS | Roteamento correto pelo proxy |

Os testes devem ser realizados sem comprometer o acesso administrativo ao servidor.

## 9. Proteção do próprio Traefik

Além do firewall, foram adotadas medidas para reduzir a superfície de exposição do contêiner.

### Dashboard administrativo

O dashboard foi desativado:

```yaml
api:
  dashboard: false
```

Isso evita disponibilizar uma interface administrativa que não é necessária para o funcionamento do proxy.

### Socket Docker

O contêiner Traefik não utiliza montagem de:

```text
/var/run/docker.sock
```

Essa decisão evita conceder ao proxy acesso direto à API administrativa do Docker.

A utilização do provedor de arquivos permite administrar as rotas sem depender desse acesso.

### Arquivos somente leitura

Os arquivos de configuração são montados como somente leitura:

```yaml
volumes:
  - /etc/traefik/traefik.yml:/etc/traefik/traefik.yml:ro
  - /etc/traefik/dynamic:/etc/traefik/dynamic:ro
```

### Limites de recursos

Foram definidos limites de memória, CPU e processos para reduzir o risco de consumo excessivo de recursos pelo contêiner.

## 10. HTTP e os riscos de comunicação sem criptografia

Durante a etapa inicial, a aplicação foi disponibilizada por HTTP, com acesso limitado às redes internas autorizadas.

Essa configuração permitiu validar:

- Resolução DNS.
- Conectividade com o servidor.
- Publicação da porta pelo Docker.
- Roteamento do Traefik.
- Comunicação com a aplicação.

Entretanto, HTTP não fornece criptografia para os dados transmitidos.

Isso significa que credenciais, cookies e informações de sessão podem ficar expostos durante o trânsito da comunicação, dependendo do contexto e das demais proteções existentes.

Por esse motivo, HTTP foi tratado como uma etapa temporária, não como o estado definitivo da arquitetura.

## 11. Evolução para HTTPS

A próxima etapa de segurança contempla a implementação de TLS.

As atividades previstas incluem:

1. Obter certificado emitido por uma autoridade certificadora aprovada.
2. Garantir que o nome DNS da aplicação esteja contemplado no certificado.
3. Instalar certificado e chave privada com permissões adequadas.
4. Configurar o entrypoint HTTPS do Traefik.
5. Habilitar TLS nas rotas correspondentes.
6. Implementar redirecionamento HTTP para HTTPS.
7. Ajustar e validar as regras de firewall para a porta 443.
8. Verificar a confiança da cadeia de certificados nos computadores clientes.
9. Definir o processo de renovação e substituição do certificado.

A conclusão dessa etapa é necessária antes de considerar a publicação definitiva da aplicação com comunicação criptografada.

## 12. Boas práticas para ambientes semelhantes

Para replicar essa arquitetura em outros ambientes, recomenda-se:

- Documentar todas as origens autorizadas.
- Evitar regras amplas de liberação.
- Revisar periodicamente as permissões.
- Manter cópias das regras antes de alterações.
- Planejar rollback para mudanças de firewall.
- Validar o comportamento após reinicializações.
- Monitorar logs do proxy e do sistema operacional.
- Manter o Traefik atualizado.
- Não publicar portas administrativas sem necessidade.
- Evitar armazenar chaves privadas em repositórios Git.
- Utilizar HTTPS para proteger a comunicação.
- Testar tanto acessos permitidos quanto bloqueados.

## 13. Conclusão

A implementação demonstrou que a publicação de aplicações por meio de um proxy reverso deve ser acompanhada de controles adequados de conectividade.

O uso da cadeia `DOCKER-USER`, associado a regras explícitas de autorização e bloqueio, permitiu estruturar o controle de acesso às portas publicadas pelo Docker.

Somado à ausência de acesso ao socket Docker, à desativação do dashboard e às montagens somente leitura, o desenho reduz a exposição desnecessária da infraestrutura.

A segurança, entretanto, deve ser tratada continuamente. A conclusão da implantação HTTPS, a revisão das regras e a validação periódica permanecem fundamentais.

**Princípio adotado: uma aplicação só deve ser publicada quando o caminho de acesso estiver definido, autorizado e validado.**

---

*Documento técnico de estudo de caso. Endereços, domínios, redes e detalhes operacionais foram generalizados para preservar a confidencialidade do ambiente original.*
