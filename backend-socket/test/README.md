# Testes do backend-socket (card #916)

Testes automáticos em JUnit 5. Eles sobem o servidor do socket numa porta livre, conectam como o frontend e como a API, e conferem as respostas das fichas.

## O que cada teste cobre

| Arquivo | Ficha | O que confere |
|---|---|---|
| `Ct19AutenticarConexaoTest` | CT19 | token válido, adulterado, vencido, usuário inexistente, usuário inativo, mensagem antes de autenticar |
| `Ct18PublicarEventoTest` | CT18 | secret errado ou ausente (fecha sem responder), campo faltando (`received:false`), evento válido (`received:true`) |
| `Ct22ChamadoAtualizadoTest` | CT22 | aviso chega no cliente dono, no técnico responsável e em todo SUPPORT e ADMIN, e não chega em outro cliente |
| `AuthUtilTest` | CT19 | conferência do token |
| `TicketEventPayloadTest` | CT18 | regras dos campos obrigatórios |
| `WebSocketUtilTest` | RFC 6455 | aperto de mão, health check e formato das mensagens |

Os testes não usam o banco Neon. O `UserDaoFalso` devolve usuários criados na memória.

## Como rodar no seu computador

Precisa do Java 21 e do arquivo `junit-platform-console-standalone-1.10.2.jar`, baixado em
https://repo1.maven.org/maven2/org/junit/platform/junit-platform-console-standalone/1.10.2/

Dentro da pasta `backend-socket` (Linux, Mac ou Git Bash):

```
mkdir -p out/main out/test
javac -encoding UTF-8 -cp "lib/*" -d out/main $(find src -name "*.java")
javac -encoding UTF-8 -cp "out/main:lib/*:CAMINHO/junit-platform-console-standalone-1.10.2.jar" -d out/test $(find test -name "*.java")
java -jar CAMINHO/junit-platform-console-standalone-1.10.2.jar execute --class-path "out/main:out/test:lib/gson.jar:lib/postgresql.jar" --select-package com.helpdesk.socket
```

No Windows (PowerShell) o separador do classpath é `;` no lugar de `:`, e os arquivos `.java` precisam ser listados com `Get-ChildItem -Recurse -Filter *.java`.

O GitHub Actions faz esses mesmos passos sozinho a cada PR (arquivo `.github/workflows/testes.yml`).
