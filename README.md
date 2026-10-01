# Tarefa 2: Relatório de Análise SAST e DAST (GoFood)
**Engenharia de Sistemas Distribuídos — UFPB**
**Aluno:** Leandeson Pinheiro Santos de Araújo

---

## 1. SAST (Static Application Security Testing)
**Ferramentas de Referência:** Semgrep / SonarQube  
**Alvo da Análise:** Código-fonte (Backend em Node.js e Frontend em React)

### Antes das Correções (Código Vulnerável)
A análise estática do código-fonte original aponta falhas estruturais críticas no momento da compilação/escrita:

*   **Hardcoded Secrets (CWE-798):** A chave de assinatura `"gofood2024secret"` encontra-se em texto limpo no ficheiro `auth.js`, acessível a qualquer utilizador com leitura do repositório.
*   **SQL Injection (CWE-89):** Identificado o uso de literais de modelo (template strings) não higienizadas em consultas SQL (ex: `WHERE email = '${email}'`) e a concatenação direta da variável `req.params.id`.
*   **Cross-Site Scripting - XSS (CWE-79):** Detetada a utilização da propriedade `dangerouslySetInnerHTML` no componente React (`App.jsx`) sem o isolamento ou limpeza prévia das strings introduzidas.
*   **Ausência de Validação de Input:** Ausência de tipagem e verificação de limites numéricos nos dados recebidos para a criação de pedidos (ex: cálculo cego de preços com base no lado do cliente).

### Depois das Correções (Código Seguro)
A varredura estática no código refatorado não deteta vulnerabilidades críticas:

*   **Gestão de Segredos:** As chaves JWT e credenciais de banco de dados foram isoladas em variáveis de ambiente (`process.env.JWT_ACCESS_SECRET`).
*   **Consultas Seguras:** O motor de análise confirma a utilização exclusiva de consultas parametrizadas (`$1`, `$2`) no driver de ligação ao PostgreSQL.
*   **Sanitização de Input/Output:** Identificada a implementação do pacote `DOMPurify.sanitize()` antes da renderização do DOM e a validação estrita de esquemas de dados com a biblioteca `Zod` nas rotas do backend.

---

## 2. DAST (Dynamic Application Security Testing)
**Ferramentas de Referência:** OWASP ZAP (Zed Attack Proxy) / Burp Suite  
**Alvo da Análise:** Aplicação em execução (Endpoints da API HTTP e renderização no Browser)

### Antes das Correções (Aplicação Vulnerável)
A varredura dinâmica, enviando *payloads* reais contra o servidor na porta 3000, reportou alertas críticos de configuração e resposta HTTP:

*   **Ausência de Rate Limiting:** A ferramenta conseguiu enviar 5.000 requisições consecutivas para o endpoint `/login` em poucos segundos, atestando a total suscetibilidade a ataques de Força Bruta e sobrecarga (*DoS*).
*   **CORS Excessivamente Permissivo:** O servidor respondeu positivamente a requisições forjadas (*Cross-Origin*) vindo de domínios maliciosos, pois o cabeçalho `Access-Control-Allow-Origin` retornava `*`.
*   **Information Disclosure (Vazamento de Informação):** Ao injetar um JSON malformatado e ultrapassar os limites físicos de memória, a API retornou o *stack trace* completo do erro no corpo da resposta (Status 500), revelando a infraestrutura de diretórios interna do servidor.
*   **Vulnerabilidade na Gestão de Sessões:** A inspeção do tráfego demonstrou que o token JWT ficava armazenado no `localStorage` do navegador, facilmente extraível via chamadas JavaScript pela consola (`window.localStorage`).

### Depois das Correções (Aplicação Segura)
Após o *deploy* dos *middlewares* de segurança e novas políticas de rede, a varredura dinâmica validou a eficácia das defesas perimetrais:

*   **Bloqueio de Força Bruta:** O ZAP Proxy começou a receber o código HTTP `429 Too Many Requests` após a 5ª tentativa falhada no endpoint de login, atestando o funcionamento do `express-rate-limit`.
*   **Defesa de Interface (Security Headers):** O *spider* da varredura identificou a presença dos cabeçalhos gerados pelo `Helmet` (ex: `Strict-Transport-Security`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`), bloqueando tentativas de *Clickjacking*.
*   **Isolamento de Sessão Segura:** A API passou a definir o estado da sessão exclusivamente via cabeçalhos `Set-Cookie`, validados pelo OWASP ZAP como seguros graças à presença conjunta das *flags* `HttpOnly`, `Secure` e `SameSite=Strict`.
*   **Filtragem de Anomalias:** Requisições contendo JSONs superiores a 16kb ou objetos mal estruturados foram sumariamente rejeitadas pelo *parser* do Express (com Status 400), sem gerar falhas em cadeia no backend.
