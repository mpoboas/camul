# Moodle + MCP — Infraestrutura de projeto

Stack Docker Compose com Moodle 5.x (`elestio/moodle:latest`) + MariaDB 11.4, com o plugin **MCP (Model Context Protocol)** instalado para permitir que agentes de IA interajam com o Moodle através de um endpoint padronizado (JSON-RPC 2.0).

Projeto académico — este repositório documenta a arquitetura, o processo de setup e os problemas reais encontrados ao longo do caminho (úteis para relatório/apresentação).

---

## 1. Arquitetura

```
Internet
   │
   │ HTTPS
   ▼
Reverse proxy / gateway (termina TLS, faz forward para a VM)
   │
   │ HTTP (porta mapeada)
   ▼
VM Ubuntu 24.04 (Docker 27.x, containerd, runc)
   │
   ├── container "moodle"      (elestio/moodle:latest)   :80  → host :<MOODLE_HOST_PORT>
   ├── container "moodle-db"   (mariadb:11.4)             :3306 (rede interna)
   │
   └── rede docker bridge (liga moodle ↔ moodle-db)
```

O stack é independente de qual gestor de containers é usado por cima (Portainer, CLI direto, etc.) — é um `docker-compose.yml` standard.

---

## 2. Pré-requisitos

- Docker Engine 24+ e Docker Compose v2 (`docker compose`, não `docker-compose`)
- Uma VM/servidor com acesso à internet de saída (necessário para instalar plugins adicionais — ver secção 6)
- Um domínio/URL público (ou pelo menos um hostname resolvível) apontado para a porta onde o Moodle vai ficar exposto — **obrigatório**, ver `MOODLE_HOST` abaixo

---

## 3. Setup

1. Copiar `.env.example` para `.env` e preencher os valores (ver tabela na secção 4):
   ```bash
   cp .env.example .env
   ```
2. Subir o stack:
   ```bash
   docker compose up -d
   ```
3. Acompanhar a instalação (pode demorar 1-2 minutos na primeira vez):
   ```bash
   docker compose logs -f moodle
   ```
   A instalação está completa quando aparece `Installation completed successfully.` seguido do arranque do Apache.
4. Aceder a `${MOODLE_HOST}` no browser e fazer login com `${MOODLE_ADMIN_USER}` / `${MOODLE_ADMIN_PASSWORD}`.

---

## 4. Variáveis de ambiente (`.env`)

Ver [`.env.example`](.env.example) para o template completo. Resumo:

| Variável | Obrigatória | Descrição |
|---|---|---|
| `MYSQL_ROOT_PASSWORD` | Sim | Password root do MariaDB |
| `MYSQL_DATABASE` | Não (default `moodle`) | Nome da base de dados |
| `MYSQL_USER` | Não (default `moodle_user`) | Utilizador da BD do Moodle |
| `MYSQL_PASSWORD` | Sim | Password desse utilizador |
| `MOODLE_ADMIN_USER` | Não (default `admin`) | Username do admin do Moodle |
| `MOODLE_ADMIN_PASSWORD` | Sim | Password do admin |
| `MOODLE_SITE_NAME` | Não | Nome do site |
| `MOODLE_HOST` | **Sim, crítico** | URL pública completa (`https://...`), incluindo porta se não for 443. Ver nota abaixo. |
| `MOODLE_HOST_PORT` | Não (default `2229`) | Porta do **host** mapeada para o container (o Apache interno corre sempre na porta 80) |

> ⚠️ **`MOODLE_HOST` é crítico.** Sem esta variável, o Moodle instala-se com `wwwroot=http://localhost`, o que parte todos os links internos, redirects de login, etc. Tem de ser o URL público real e definitivo — mudar depois exige editar `config.php` manualmente ou reinstalar.

> ⚠️ **Porta interna é sempre 80, não 8080.** Esta imagem (`elestio/moodle:latest`) corre Apache na porta 80 standard — não confundir com imagens estilo Bitnami que usam 8080. O `docker-compose.yml` já reflete isto corretamente (`"${MOODLE_HOST_PORT}:80"`); se copiares de outro sítio um exemplo com `:8080`, corrige.

---

## 5. Caminhos reais dentro da imagem (para quem for mexer nos volumes)

Confirmado a partir do próprio script de instalação da imagem (`/docker-entrypoint.d/20-moodle-install.sh`):

| Conteúdo | Caminho |
|---|---|
| Código do Moodle (+ `config.php`) | `/var/www/html` |
| Document root do Apache | `/var/www/html/public` |
| Dados de utilizadores / ficheiros | `/var/moodledata` |

**Não** `/bitnami/moodle` / `/bitnami/moodledata` — essa é a convenção de imagens Bitnami, não aplicável aqui. Confirma sempre os caminhos reais de qualquer imagem lendo o `docker-entrypoint.d/*` antes de assumir convenções de outra imagem.

---

## 6. Troubleshooting — problemas reais encontrados

Estas são lições aprendidas durante o desenvolvimento deste projeto. Generalizadas para qualquer setup Docker semelhante:

### `docker exec` / healthchecks ficam pendurados ou falham com `fork/exec /proc/self/exe`
Sintoma de dessincronização entre `dockerd` e `containerd` (por exemplo depois de uma atualização de pacotes com containers a correr). Ordem de tentativas:
1. `systemctl restart containerd docker`
2. Se não resolver: testar `runc exec` diretamente (bypass ao dockerd) — se funcionar instantaneamente, o problema está especificamente na camada `dockerd`, não no runtime.
3. Último recurso, mas o que realmente resolve de forma fiável: `systemctl reboot` à VM. Containers com `restart: always` voltam sozinhos.

### Instalação falha com erro de "unicode" mesmo com a collation certa no MariaDB
Não confiar cegamente na mensagem — pode não ser a collation. Garantir que o Moodle só tenta ligar-se à BD depois desta estar **genuinamente pronta** (não só a aceitar TCP): usar `healthcheck` no serviço da BD + `depends_on: condition: service_healthy` no Moodle, em vez de um `depends_on` simples ou espera fixa.

### Cópia de volume incompleta
Se depois de corrigir os volumes o Moodle falhar com ficheiros PHP em falta (`Failed opening required ...`), suspeitar de uma cópia interrompida do conteúdo da imagem para um volume novo (acontece se `containerd` estiver instável — ver ponto acima). Confirmar comparando a contagem de ficheiros do volume com a imagem "limpa" (`docker run --rm --entrypoint sh <imagem> -c 'find /caminho | wc -l'`), apagar o volume e recriar.

### Site atrás de proxy: `sslproxy` vs `reverseproxy`
Se o site correr atrás de um proxy que só termina TLS (sem injetar headers `X-Forwarded-*`), usar **apenas** `$CFG->sslproxy = true;`. Ativar também `$CFG->reverseproxy = true;` nesse cenário pode **bloquear o acesso** com o erro "Reverse proxy enabled so the server cannot be accessed directly" — essa flag depende de headers de forwarding que o proxy pode não estar a enviar.

### Container sem acesso à internet (mas o host tem)
Se precisares de instalar algo de dentro do container (`git clone`, `curl` para fora) e falhar por timeout mesmo com o host a ter internet, é provável que seja uma política de rede do ambiente (NAT/forward bloqueado para tráfego originado em containers). Solução prática: fazer o download no **host** e escrever diretamente no caminho do volume Docker (`docker volume inspect <nome> --format '{{.Mountpoint}}'` dá o caminho no filesystem do host).

---

## 7. MCP (Model Context Protocol)

### O que é
Plugin de terceiros (`webservice_mcp`) que expõe as funções de web service do Moodle como ferramentas MCP (JSON-RPC 2.0), permitindo que agentes de IA (Claude, ChatGPT, etc.) interajam com o Moodle de forma padronizada.

Repositório: https://github.com/onbirdev/moodle-webservice_mcp

> Nota: o **Moodle AI Subsystem** nativo (`admin/ai.php`) é uma coisa diferente — liga o Moodle a providers de IA (OpenAI, Gemini, etc.) para funcionalidades *dentro* da UI do Moodle. Não é o mesmo que o MCP.

### Instalação

O container Moodle normalmente não tem acesso à internet de saída (ver secção 6). Instala-se a partir do host:

```bash
# 1. Descobrir o mountpoint real do volume de código
docker volume inspect <projeto>_moodle_www --format '{{.Mountpoint}}'

# 2. Clonar diretamente nesse caminho, no subdiretório webservice/
git clone https://github.com/onbirdev/moodle-webservice_mcp.git \
  <mountpoint>/public/webservice/mcp

# 3. Corrigir ownership (o Apache corre como www-data)
chown -R www-data:www-data <mountpoint>/public/webservice/mcp
rm -rf <mountpoint>/public/webservice/mcp/.git   # opcional, limpeza

# 4. Registar o plugin na base de dados
docker exec -u www-data <container_moodle> php /var/www/html/admin/cli/upgrade.php --non-interactive
```

### Ativação

```bash
docker exec -u www-data <container_moodle> php /var/www/html/admin/cli/cfg.php --name=enablewebservices --set=1
docker exec -u www-data <container_moodle> php /var/www/html/admin/cli/cfg.php --name=webserviceprotocols --set=mcp,rest
```

### Configuração (via interface web, Site administration → Server → Web services)

1. **External services → Add** — criar um serviço (ex: "MCP Service"), com "Authorised users only" conforme o teu caso de uso.
2. Adicionar as funções que queres expor a esse serviço.
3. Autorizar os utilizadores que vão usar esse serviço (se "Authorised users only" estiver ativo).
4. **Manage tokens → Create token** — escolher o utilizador + o serviço criado.

### ⚠️ Gotcha importante: a capacidade `webservice/mcp:use`

O plugin define a capacidade `webservice/mcp:use` **sem a atribuir a nenhum role por omissão**. Sem este passo, **todas** as chamadas MCP falham com um erro genérico de "Access control exception", independentemente das funções no serviço.

**Atribuir esta capacidade ao role "Authenticated user"** (não aos roles específicos como Teacher/Student — esses são normalmente atribuídos a nível de curso, e a verificação inicial do MCP acontece a nível de sistema; uma atribuição de role a nível de curso não "sobe" para o contexto de sistema no Moodle). Via **Site administration → Users → Permissions → Define roles → Authenticated user**, adicionar `webservice/mcp:use` = Allow.

A diferenciação real de permissões (o que cada utilizador consegue mesmo fazer via MCP) continua a acontecer normalmente através das capacidades específicas de cada função chamada — o Moodle valida-as em cada `tools/call` exatamente como validaria um pedido feito através da interface web.

### Modelo de permissões: serviço vs. token

A lista de funções expostas é uma propriedade do **serviço externo**, não do token individual. Não é possível ter dois tokens no mesmo serviço com conjuntos de funções diferentes. Para acessos com âmbitos diferentes, criar **serviços separados**, cada um com o seu subconjunto de funções, e emitir tokens contra o serviço apropriado.

### Endpoint e exemplos

```
POST https://<MOODLE_HOST>/webservice/mcp/server.php?wstoken=<TOKEN>
Content-Type: application/json
```

Ou com o token no header:
```
Authorization: Bearer <TOKEN>
```

**Listar ferramentas disponíveis:**
```bash
curl -X POST "https://<MOODLE_HOST>/webservice/mcp/server.php?wstoken=<TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","params":{},"id":1}'
```

**Invocar uma ferramenta:**
```bash
curl -X POST "https://<MOODLE_HOST>/webservice/mcp/server.php?wstoken=<TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc":"2.0",
    "method":"tools/call",
    "params":{"name":"core_webservice_get_site_info","arguments":{}},
    "id":2
  }'
```

Métodos MCP suportados: `initialize`, `notifications/initialized`, `ping`, `tools/list`, `tools/call`.

---

## 8. Segurança — antes de ires além de testes

- Nunca comitar `.env` (já está no `.gitignore`) nem tokens/passwords reais neste repositório.
- Tokens ligados a contas com role Manager/Admin têm acesso equivalente a esse utilizador em **todas** as 755+ funções, se o serviço as incluir todas — usar com cuidado, preferencialmente só para testes controlados.
- Para uso real, criar serviços com subconjuntos de funções revistos manualmente (evitar incluir funções destrutivas — `*_delete_*`, `*_delete_users`, etc. — em serviços cujo token não precisa mesmo delas).
- Rotacionar qualquer credencial que tenha sido partilhada informalmente (chat, email) antes de considerar o ambiente "seguro".
