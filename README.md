# Documentar-o-Projeto
Aluno Angelo Dos Santos Monteiro

# Código comentado:

import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpHandler;
import com.sun.net.httpserver.HttpServer;

import java.io.*;
import java.net.InetSocketAddress;
import java.net.URI;
import java.nio.charset.StandardCharsets;
import java.nio.file.*;
import java.util.*;

public class App {

    // Configurações principais do servidor e "banco" em memória
    static final int PORT = 8080;
    static final String CSV = "data_tasks.csv";
    static final int MAX = 5000;

    // Estrutura de dados em arrays para armazenar tarefas
    static String[] ids = new String[MAX];
    static String[] titulos = new String[MAX];
    static String[] descrs = new String[MAX];
    static int[] status = new int[MAX];
    static long[] criados = new long[MAX];
    static int n = 0;

    public static void main(String[] args) throws Exception {
        carregar(); // lê o CSV existente e carrega para memória

        // Cria e configura servidor HTTP
        HttpServer server = HttpServer.create(new InetSocketAddress(PORT), 0);

        // Rota raiz "/" → devolve o HTML com a interface do Kanban
        server.createContext("/", new RootHandler());

        // Rota "/api/tasks" → devolve/manipula as tarefas (JSON)
        server.createContext("/api/tasks", new ApiTasksHandler());

        server.setExecutor(null); // usa executor padrão
        System.out.println("Servindo em http://localhost:" + PORT);
        server.start(); // inicia servidor
    }


    // HANDLER PARA A PÁGINA HTML
    static class RootHandler implements HttpHandler {
        @Override public void handle(HttpExchange ex) throws IOException {

            // Só permite GET, outros métodos retornam 405 (não permitido)
            if (!"GET".equalsIgnoreCase(ex.getRequestMethod())) { send(ex, 405, ""); return; }

            // Devolve a página HTML definida em INDEX_HTML
            byte[] body = INDEX_HTML.getBytes(StandardCharsets.UTF_8);
            ex.getResponseHeaders().set("Content-Type", "text/html; charset=utf-8");
            ex.sendResponseHeaders(200, body.length);
            try (OutputStream os = ex.getResponseBody()) { os.write(body); }
        }
    }
    // HANDLER PARA A API REST (CRUD de tarefas)
    static class ApiTasksHandler implements HttpHandler {
        @Override public void handle(HttpExchange ex) throws IOException {
            String method = ex.getRequestMethod();
            URI uri = ex.getRequestURI();
            String path = uri.getPath();

            try {
                // GET /api/tasks → lista todas as tarefas em JSON
                if ("GET".equals(method) && "/api/tasks".equals(path)) {
                    sendJson(ex, 200, listarJSON());
                    return;
                }

                // POST /api/tasks → cria nova tarefa
                if ("POST".equals(method) && "/api/tasks".equals(path)) {
                    String body = new String(ex.getRequestBody().readAllBytes(), StandardCharsets.UTF_8);
                    String titulo = jsonGet(body, "titulo");
                    String descricao = jsonGet(body, "descricao");
                    if (titulo == null || titulo.isBlank()) {
                        sendJson(ex, 400, "{\"error\":\"titulo obrigatório\"}");
                        return;
                    }

                    // cria e salva tarefa
                    Map<String, Object> t = criar(titulo, descricao == null ? "" : descricao);
                    salvar();
                    sendJson(ex, 200, toJsonTask(t));
                    return;
                }

                // PATCH /api/tasks/{id}/status → altera status de uma tarefa
                if ("PATCH".equals(method) && path.startsWith("/api/tasks/") && path.endsWith("/status")) {

                    String id = path.substring("/api/tasks/".length(), path.length() - "/status".length());
                    String body = new String(ex.getRequestBody().readAllBytes(), StandardCharsets.UTF_8);
                    String stStr = jsonGet(body, "status");
                    if (stStr == null) { sendJson(ex, 400, "{\"error\":\"status ausente\"}"); return; }
                    int st = clampStatus(parseIntSafe(stStr, 0));
                    int i = findIdxById(id);
                    if (i < 0) { sendJson(ex, 404, "{\"error\":\"not found\"}"); return; }
                    status[i] = st;
                    salvar();
                    sendJson(ex, 200, toJsonTask(mapOf(i)));
                    return;
                }

                // DELETE /api/tasks/{id} → exclui tarefa
                if ("DELETE".equals(method) && path.startsWith("/api/tasks/")) {
                    String id = path.substring("/api/tasks/".length());
                    int i = findIdxById(id);
                    if (i < 0) { sendJson(ex, 404, "{\"error\":\"not found\"}"); return; }

                    // reorganiza arrays (remove elemento e puxa os seguintes para trás)
                    for (int k = i; k < n - 1; k++) {
                        ids[k] = ids[k+1]; titulos[k] = titulos[k+1]; descrs[k] = descrs[k+1];
                        status[k] = status[k+1]; criados[k] = criados[k+1];
                    }
                    n--;
                    salvar();
                    sendJson(ex, 204, "");
                    return;
                }

                // Se nenhuma rota foi atendida → 404
                send(ex, 404, "");
            } catch (Exception e) {
                e.printStackTrace();
                sendJson(ex, 500, "{\"error\":\"server\"}");
            }
        }
    }


    // Página principal com formulário + JS que consome a API
    static final String INDEX_HTML = """
<!doctype html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Kanban Local (sem framework)</title>
<style>
  :root{--bg:#f6f7fb;--card:#fff;--muted:#666;}
  *{box-sizing:border-box} body{margin:0;font:16px system-ui,Segoe UI,Roboto,Arial;background:var(--bg)}
  header{background:#111;color:#fff;padding:12px 16px}
  .wrap{max-width:1100px;margin:0 auto;padding:16px}
  form{display:flex;gap:8px;flex-wrap:wrap;margin:12px 0}
  input,textarea,button,select{border:1px solid #ddd;border-radius:10px;padding:10px;font:inherit}
  textarea{min-width:260px;min-height:40px}
  button{cursor:pointer}
  .board{display:grid;grid-template-columns:repeat(3,1fr);gap:16px}
  .col{background:var(--card);border-radius:14px;box-shadow:0 6px 16px rgba(0,0,0,.06);padding:12px}
  .col h2{margin:6px 4px 10px}
  .task{border:1px solid #eee;border-radius:12px;background:#fafafa;margin:8px 0;padding:10px}
  .row{display:flex;gap:8px;flex-wrap:wrap;align-items:center}
  small{color:var(--muted)}
  .pill{font-size:.75rem;padding:2px 8px;border-radius:999px;background:#eef;border:1px solid #dde}
</style>
</head>
<body>
<header><b>Kanban Local</b> — Gestão de Atividades</header>
<div class="wrap">

  <h3>Nova tarefa</h3>
  <form id="f">
    <input id="t" placeholder="Título" required>
    <textarea id="d" placeholder="Descrição (opcional)"></textarea>
    <button>Adicionar</button>
  </form>

  <div class="board">
    <div class="col"><h2>To-Do</h2><div id="todo"></div></div>
    <div class="col"><h2>Doing</h2><div id="doing"></div></div>
    <div class="col"><h2>Done</h2><div id="done"></div></div>
  </div>

</div>

<script>
const API = "/api/tasks";

async function listar(){
  const r = await fetch(API);
  const data = await r.json();
  render(data);
}

function el(html){
  const t = document.createElement('template');
  t.innerHTML = html.trim();
  return t.content.firstChild;
}

// CORRIGIDO: função sem erro de sintaxe
function escapeHtml(s){
  return s.replace(/[&<>\"']/g, c => ({
    '&':'&amp;',
    '<':'&lt;',
    '>':'&gt;',
    '\"':'&quot;',
    \"'\":'&#039;'
  }[c]));
}

function card(t){
  const div = el(`<div class="task">
      <strong>${escapeHtml(t.titulo)}</strong>
      <div class="row">
        <span class="pill">${['TODO','DOING','DONE'][t.status] || t.status}</span>
        <small>criado: ${new Date(t.criadoEm).toLocaleString()}</small>
      </div>
      <p>${escapeHtml(t.descricao||'')}</p>
      <div class="row">
        ${t.status!==0?'<button data-prev>◀</button>':''}
        ${t.status!==2?'<button data-next>▶</button>':''}
        <button data-del>Excluir</button>
      </div>
  </div>`);

  if(div.querySelector('[data-prev]')){
    div.querySelector('[data-prev]').onclick=()=>mover(t.id, t.status===2?1:0);
  }
  if(div.querySelector('[data-next]')){
    div.querySelector('[data-next]').onclick=()=>mover(t.id, t.status===0?1:2);
  }
  div.querySelector('[data-del]').onclick=()=>excluir(t.id);
  return div;
}

function render(arr){
  ['todo','doing','done'].forEach(id=>document.getElementById(id).innerHTML='');
  arr.filter(x=>x.status===0).forEach(x=>document.getElementById('todo').appendChild(card(x)));
  arr.filter(x=>x.status===1).forEach(x=>document.getElementById('doing').appendChild(card(x)));
  arr.filter(x=>x.status===2).forEach(x=>document.getElementById('done').appendChild(card(x)));
}

document.getElementById('f').onsubmit=async (e)=>{
  e.preventDefault();
  const titulo = document.getElementById('t').value.trim();
  const descricao = document.getElementById('d').value.trim();
  if(!titulo) return;
  await fetch(API,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({titulo,descricao})});
  e.target.reset(); listar();
};

async function mover(id,status){
  await fetch(`${API}/${id}/status`,{method:'PATCH',headers:{'Content-Type':'application/json'},body:JSON.stringify({status})});
  listar();
}
async function excluir(id){
  await fetch(`${API}/${id}`,{method:'DELETE'}); listar();
}

listar();
</script>
</body>
</html>
""";

    // Carrega tarefas de arquivo CSV para memória
    static void carregar() {
        n = 0;
        Path p = Paths.get(CSV);
        if (!Files.exists(p)) return;
        try (BufferedReader br = Files.newBufferedReader(p, StandardCharsets.UTF_8)) {
            String line;
            while ((line = br.readLine()) != null) {
                if (line.isBlank() || line.startsWith("id;")) continue;
                String[] a = splitCsv(line);
                if (a.length < 5) continue;
                if (n >= MAX) break;
                ids[n] = a[0];
                titulos[n] = a[1];
                descrs[n] = a[2];
                status[n] = clampStatus(parseIntSafe(a[3], 0));
                criados[n] = parseLongSafe(a[4], System.currentTimeMillis());
                n++;
            }
            //Tratamento de exceções em caso de falha no carregamento
        } catch (IOException e) {
            System.out.println("Falha ao ler CSV: " + e.getMessage());
        }
    }

    // Salva as tarefas em CSV
    static void salvar() {
        Path p = Paths.get(CSV);
        try {
            if (p.getParent()!=null) Files.createDirectories(p.getParent());
            try (BufferedWriter bw = Files.newBufferedWriter(p, StandardCharsets.UTF_8,
                    StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING)) {
                bw.write("id;titulo;descricao;status;criadoEm\n");
                for (int i = 0; i < n; i++) {
                    bw.write(esc(ids[i]) + ";" + esc(titulos[i]) + ";" + esc(descrs[i]) + ";"
                            + status[i] + ";" + criados[i] + "\n");
                }
            }
        } catch (IOException e) {
            System.out.println("Falha ao salvar CSV: " + e.getMessage());
        }
    }

    // Cria uma nova tarefa em memória
    static Map<String, Object> criar(String titulo, String descr) {
        if (n >= MAX) throw new RuntimeException("Capacidade cheia");
        String id = UUID.randomUUID().toString().substring(0,8);
        ids[n]=id; titulos[n]=titulo; descrs[n]=descr; status[n]=0; criados[n]=System.currentTimeMillis();
        n++;
        return mapOf(n-1);
    }

    // Busca índice de tarefa pelo ID
    static int findIdxById(String id){
        for (int i=0;i<n;i++) if (ids[i].equals(id)) return i;
        return -1;
    }

    // Converte tarefa de índice i em um Map (estrutura chave/valor)
    static Map<String,Object> mapOf(int i){
        Map<String,Object> m=new LinkedHashMap<>();
        m.put("id", ids[i]); m.put("titulo", titulos[i]); m.put("descricao", descrs[i]);
        m.put("status", status[i]); m.put("criadoEm", criados[i]);
        return m;
    }

    // Retorna todas as tarefas em JSON
    static String listarJSON(){
        StringBuilder sb = new StringBuilder("[");
        for (int i=0;i<n;i++){
            if (i>0) sb.append(',');
            sb.append(toJsonTask(mapOf(i)));
        }
        sb.append(']');
        return sb.toString();
    }

    // Converte tarefa isolada para JSON
    static String toJsonTask(Map<String,Object> t){
        return "{\"id\":\""+jsonEsc((String)t.get("id"))+"\"," +
                "\"titulo\":\""+jsonEsc((String)t.get("titulo"))+"\"," +
                "\"descricao\":\""+jsonEsc((String)t.get("descricao"))+"\"," +
                "\"status\":" + t.get("status") + "," +
                "\"criadoEm\":" + t.get("criadoEm") + "}";
    }

    // Extrai o valor de uma chave dentro de um JSON simples
    static String jsonGet(String body, String key){
        if (body == null) return null;
        String s = body.trim();

        // Remove as chaves { } externas
        if (s.startsWith("{")) s = s.substring(1);
        if (s.endsWith("}")) s = s.substring(0, s.length()-1);

        // Divide em pares chave:valor
        List<String> parts = new ArrayList<>();
        StringBuilder cur = new StringBuilder();
        boolean inQ = false;
        for (int i=0;i<s.length();i++){
            char c = s.charAt(i);
            if (c=='"' && (i==0 || s.charAt(i-1)!='\\')) inQ = !inQ;
            if (c==',' && !inQ){ parts.add(cur.toString()); cur.setLength(0); }
            else cur.append(c);
        }
        if (cur.length()>0) parts.add(cur.toString());

        // Procura a chave desejada dentro da lista
        for (String kv : parts){
            int i = kv.indexOf(':');
            if (i<=0) continue;
            String k = kv.substring(0,i).trim();
            String v = kv.substring(i+1).trim();
            k = stripQuotes(k);
            if (key.equals(k)){
                v = stripQuotes(v);
                return v;
            }
        }
        return null;
    }

    // Remove aspas de uma string JSON e trata caracteres escapados
    static String stripQuotes(String s){
        s = s.trim();
        if (s.startsWith("\"") && s.endsWith("\"")) {
            s = s.substring(1, s.length()-1).replace("\\\"", "\"");
        }
        return s;
    }

    // Envia resposta HTTP genérica
    static void send(HttpExchange ex, int code, String body) throws IOException {
        byte[] bytes = body.getBytes(StandardCharsets.UTF_8);
        ex.sendResponseHeaders(code, bytes.length);
        try (OutputStream os = ex.getResponseBody()) { os.write(bytes); }
    }

    // Envia resposta HTTP com cabeçalho "application/json"
    static void sendJson(HttpExchange ex, int code, String body) throws IOException {
        ex.getResponseHeaders().set("Content-Type","application/json; charset=utf-8");
        send(ex, code, body==null?"":body);
    }

    // Escapa valores para salvar em CSV
    static String esc(String s){
        if (s==null) return "";
        if (s.contains(";") || s.contains("\"") || s.contains("\n")) return "\"" + s.replace("\"", "\"\"") + "\"";
        return s;
    }

    // Divide uma linha CSV em campos
    static String[] splitCsv(String line){
        List<String> out = new ArrayList<>();
        StringBuilder cur = new StringBuilder();
        boolean inQ = false;
        for (int i=0;i<line.length();i++){
            char c = line.charAt(i);
            if (inQ){
                if (c=='"'){
                    if (i+1<line.length() && line.charAt(i+1)=='"'){ cur.append('"'); i++; }
                    else inQ=false;
                } else cur.append(c);
            } else {
                if (c==';'){ out.add(cur.toString()); cur.setLength(0); }
                else if (c=='"'){ inQ=true; }
                else cur.append(c);
            }
        }
        out.add(cur.toString());
        return out.toArray(new String[0]);
    }
    static String jsonEsc(String s){
        if (s==null) return "";
        return s.replace("\\","\\\\").replace("\"","\\\"").replace("\n","\\n").replace("\r","");
    }

    // Garante que o status esteja no intervalo 0-2 (TODO, DOING, DONE)
    static int clampStatus(int s){ return Math.max(0, Math.min(2, s)); }

    // Converte string em inteiro de forma segura, retorna valor padrão se falhar
    static int parseIntSafe(String s, int def){ try { return Integer.parseInt(s.trim()); } catch(Exception e){ return def; } }

    // Converte string em long de forma segura, retorna valor padrão se falhar
    static long parseLongSafe(String s, long def){ try { return Long.parseLong(s.trim()); } catch(Exception e){ return def; } }
}

# Avaliação do projeto

O sistema desenvolvido é um protótipo de um quadro Kanban. Ele funciona bem para aprendizado, pois mostra como criar um servidor HTTP básico, manipular requisições, respostas e salvar dados em um arquivo CSV. A interface HTML é simples, mas permite visualizar e organizar tarefas em colunas “To-Do”, “Doing” e “Done”. Entre os pontos positivos, pode-se destacar a simplicidade, já que o projeto ajuda a entender fundamentos de servidores e manipulação de dados em Java. Também é interessante o fato dos dados poderem ser salvos no arquivo, evitando que seja perdidos ao encerrar o programa. No entanto, o projeto tem várias limitações. O uso de arrays fixos limita a quantidade de tarefas possíveis e não é escalável. O tratamento de JSON e CSV que foi feito de forma manual, pode gerar erros e tornar o sistema menos seguro. Outro ponto é a ausência de autenticação, já que qualquer usuário pode acessar e alterar as tarefas ou até mesmo excluir as tarefas. Por fim, o sistema funciona apenas localmente, o que dificulta o uso em equipe.

# Proposta de Sistema Substituto

O back-end poderia ser refeito em frameworks como Spring Boot, no caso de continuar em Java, ou em Node.js, caso tenha que ser mais leve. Isso permitiria criar uma API mais segura e escalável, além de facilitar a manutenção. No lugar do arquivo CSV, seria melhor adotar um banco de dados, que poderia ser relacional, como PostgreSQL ou MySQL, ou não relacional, como MongoDB. Isso possibilitaria o armazenamento organizado, seguro e sem limite fixo de tarefas. A interface, que atualmente é apenas um HTML simples, poderia ser feita separadamente com ferramentas como React, tornando-a mais interativa e moderna. Funcionalidades como arrastar e soltar as tarefas entre colunas seriam possíveis, além de deixar a aplicação mais amigável. Também seria necessário implementar autenticação, para que cada usuário tivesse seu próprio quadro de tarefas e houvesse controle de acesso. Com isso, o sistema deixaria de ser apenas local e poderia ser hospedado em servidores na nuvem, possibilitando uso em equipe. Por fim, outras funcionalidades poderiam ser adicionadas, como categorias de tarefas, etiquetas de cores, notificações e exportação de relatórios em PDF ou Excel.
