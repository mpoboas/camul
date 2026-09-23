# CLAUDE.md

Instruções para agentes Claude Code que trabalhem neste repositório. Lê isto antes de tocar em qualquer coisa relacionada com o servidor Moodle.

## O que é este projeto

Projeto académico: Moodle (`elestio/moodle:latest`) + MariaDB via Docker Compose, com o plugin `webservice_mcp` instalado para expor o Moodle a agentes de IA via Model Context Protocol. Ver [`README.md`](README.md) para a arquitetura completa e [`docker-compose.yml`](docker-compose.yml) / [`.env.example`](.env.example) para a configuração.

## Credenciais e acesso ao servidor

**Este repositório não contém nenhuma credencial real.** As credenciais reais (SSH do servidor, password admin do Moodle, tokens MCP) estão fora do git — pede-as a quem já tem acesso (colega do grupo) e guarda-as localmente num `.env` (já no `.gitignore`) ou num gestor de passwords. Nunca as cole diretamente nas mensagens de chat com o agente sem necessidade — se o fizeres, considera essas credenciais comprometidas e pede para serem rotacionadas.

Quando um colega te der acesso SSH ao servidor, guarda o essencial num sítio que o agente possa reler sem pedires de novo (ex: variável de ambiente local, ficheiro `.env` não commitado) em vez de reintroduzir a password a cada sessão.

## Regras de segurança para o agente

- **Nunca** commitar `.env`, chaves privadas, tokens ou passwords para o git, mesmo que peçam explicitamente "só para testar". Se um ficheiro sensível for acidentalmente criado dentro do repo, apaga-o e avisa o utilizador.
- Antes de qualquer ação destrutiva no servidor real (`docker compose down -v`, apagar volumes, `docker volume rm`, apagar cursos/utilizadores no Moodle via MCP ou SQL), confirma explicitamente com quem está a pedir — mesmo que pareça óbvio pelo contexto.
- Ao gerar tokens MCP ou atribuir capacidades/roles no Moodle, prefere o mínimo de privilégio necessário para o teste em causa. Não associes tokens de teste à conta `admin`/Manager por conveniência se o caso de uso não precisar disso.
- Se precisares de instalar algo de dentro do container `moodle` e falhar por falta de rede, não é o teu ambiente que está mal configurado — é uma limitação conhecida de rede do host (ver README secção 6). Faz o download no host e escreve diretamente no volume Docker.

## Conhecimento operacional acumulado (não repetir os mesmos erros)

Ver [`README.md`](README.md) secção 6 para o detalhe completo. Resumo rápido para referência rápida do agente:

1. **`docker exec` ou healthchecks pendurados / erros `fork/exec /proc/self/exe`** → sintoma de dessincronização `dockerd`↔`containerd`. Tentar `systemctl restart containerd docker` primeiro; se persistir, `systemctl reboot` à VM resolve de forma fiável (containers com `restart: always` voltam sozinhos). Testar `runc exec` diretamente para confirmar se o runtime em si está saudável antes de assumir que é preciso reboot.
2. **Porta interna do Apache nesta imagem é sempre 80**, não 8080 — não assumir convenções Bitnami.
3. **Caminhos de volumes reais**: `/var/www/html` (código) e `/var/moodledata` (dados) — não `/bitnami/...`. Confirmar sempre lendo `/docker-entrypoint.d/*` da imagem antes de assumir.
4. **`$CFG->reverseproxy = true` pode partir o acesso** se o proxy à frente não injetar headers `X-Forwarded-*` — usar só `sslproxy` nesse caso.
5. **MCP: a capacidade `webservice/mcp:use` não está atribuída a nenhum role por omissão.** Sem a atribuir ao role "Authenticated user" a nível de sistema, todas as chamadas MCP falham com um erro genérico de "Access control exception", independentemente do que estiver no serviço. Não perder tempo a debugar funções individuais sem confirmar primeiro que esta capacidade está atribuída.
6. **MCP: a lista de funções expostas é propriedade do serviço externo, não do token.** Para tokens com âmbitos diferentes, criar serviços separados — não hà como restringir por token dentro do mesmo serviço.
7. Antes de assumir que um erro é do "runtime"/Docker, testar sempre ao nível mais baixo possível primeiro (ex: `runc exec` direto) para isolar em que camada está o problema real.

## Fluxo de trabalho esperado

- Alterações ao `docker-compose.yml` ou `.env.example` neste repo → depois aplicar manualmente no servidor real (não há CI/CD automático a fazer deploy).
- Mudanças de configuração feitas diretamente no servidor (ex: via `docker exec`, SQL direto, `admin/cli/cfg.php`) devem ser refletidas de volta neste repositório quando fizer sentido, para o resto do grupo não perder o histórico de decisões.
- Ao investigar um problema, preferir ler os scripts de entrypoint da própria imagem (`docker run --rm --entrypoint sh <imagem> -c 'cat /docker-entrypoint.d/*'`) a assumir comportamento de outras imagens Moodle/Bitnami que já se conheçam.
