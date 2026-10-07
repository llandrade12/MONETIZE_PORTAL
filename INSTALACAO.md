# Instalação — Monetize · Área de Membros (Mentoria Suba o Padrão)

Este guia leva o portal do zero ao ar na HostGator. Siga na ordem.

## 0. Antes de começar: qual plano foi contratado?

O servidor é um programa Java que fica ligado o tempo todo. **Hospedagem compartilhada da HostGator (planos P, M, Business, Turbo) não roda Java.** É preciso um **VPS** (ou servidor dedicado).

Como descobrir (peça ao gestor que contratou, ou entre no Portal do Cliente da HostGator → *Hospedagens*):

| O que aparece no Portal do Cliente | O que fazer |
|---|---|
| "VPS" com acesso ao **EasyPanel** | Siga o **Caminho A** (Docker) |
| "VPS" com **cPanel/WHM**, ou VPS só com acesso SSH (root) | Siga o **Caminho B** |
| "Hospedagem de Sites", "Plano P/M/Business/Turbo", "Revenda" | Não serve. É preciso contratar um VPS (o mais simples, de 2 GB de memória, já atende) |

Requisitos mínimos do VPS: 1 vCPU, 2 GB de memória, 20 GB de disco, Linux (Ubuntu 22.04/24.04, AlmaLinux 8/9 ou similar).

Você também precisa de:

- um **endereço** para o portal, por exemplo `membros.seudominio.com.br`, apontando para o IP do VPS (registro tipo **A** na zona DNS do domínio);
- uma **caixa de e-mail** para o portal enviar avisos, por exemplo `portal@seudominio.com.br` (crie no painel de e-mails da HostGator e guarde a senha).

## 1. O que vem no pacote

```
portal/
  backend/                      servidor Java
    suba-portal.jar             programa pronto para rodar
    config.exemplo.properties   modelo de configuração
    compilar.sh                 gera o .jar de novo, só com o JDK
    pom.xml                     projeto Maven (mvn package)
    src/                        código-fonte, script do banco e testes
  web/                          as páginas do portal (login, aluno, mentor)
  deploy/vps/                   serviço systemd, Nginx, Apache, backup
  deploy/docker/                Dockerfile e docker-compose (EasyPanel/Docker)
  docs/                         este guia e o README técnico
```

O banco de dados **não precisa de script manual**: na primeira vez que sobe, o servidor cria as tabelas sozinho (o script está em `backend/src/main/resources/db/V001__esquema_inicial.sql`, para consulta). O sistema nasce **zerado**: nenhuma turma, aluno, aula ou tarefa.

---

## Caminho A — VPS com EasyPanel (Docker)

1. **Suba o projeto para um repositório Git privado** (GitHub, GitLab ou Bitbucket) da equipe. O EasyPanel monta a imagem a partir dele.
2. No EasyPanel, crie um **Projeto** (ex.: `mentoria`).
3. Dentro dele, crie um serviço **MySQL** (*+ Service → MySQL*). Nome do banco: `suba_portal`. Anote usuário, senha e o **host interno** que o EasyPanel mostra (algo como `mentoria_mysql`).
4. Crie um serviço **App**:
   - *Source*: o repositório Git. *Build*: **Dockerfile**, caminho `deploy/docker/Dockerfile`.
   - *Domains*: adicione `membros.seudominio.com.br`, porta do contêiner **8080**, HTTPS ligado (o EasyPanel emite o certificado).
   - *Mounts*: adicione um **Volume** com caminho `/dados/arquivos` (é onde ficam os arquivos enviados; sem isso eles se perdem a cada atualização).
   - *Environment*: cole e preencha:

     ```
     SOP_URL_PUBLICA=https://membros.seudominio.com.br
     SOP_FUSO=America/Bahia
     SOP_DB_URL=jdbc:mysql://HOST_INTERNO_DO_MYSQL:3306/suba_portal?characterEncoding=UTF-8&connectionCollation=utf8mb4_unicode_ci&useSSL=false&allowPublicKeyRetrieval=true
     SOP_DB_USUARIO=usuario_do_mysql
     SOP_DB_SENHA=senha_do_mysql
     SOP_ADMIN_NOME=Nome do Mentor
     SOP_ADMIN_EMAIL=mentor@seudominio.com.br
     SOP_ADMIN_SENHA=UmaSenhaForte123
     SOP_SMTP_HOST=mail.seudominio.com.br
     SOP_SMTP_PORTA=465
     SOP_SMTP_SEGURANCA=ssl
     SOP_SMTP_USUARIO=portal@seudominio.com.br
     SOP_SMTP_SENHA=senha_da_caixa
     SOP_SMTP_REMETENTE=portal@seudominio.com.br
     SOP_SMTP_REMETENTE_NOME=Monetize · Área de Membros
     ```
5. Clique em **Deploy**. Nos *Logs* deve aparecer `Área de Membros 1.0.0 no ar`.
6. Vá para a seção **3. Primeiro acesso**.

Sem EasyPanel, mas com Docker instalado no VPS: copie `deploy/docker/.env.exemplo` para `deploy/docker/.env`, preencha, e rode na raiz do projeto `docker compose -f deploy/docker/docker-compose.yml up -d --build`. O portal responde na porta 8080; coloque um proxy com HTTPS na frente (item B.6).

---

## Caminho B — VPS com SSH (com ou sem cPanel/WHM)

Entre no VPS por SSH como `root` (os dados de acesso estão no Portal do Cliente da HostGator).

### B.1 Instalar o Java 17

```bash
# Ubuntu / Debian
apt update && apt install -y openjdk-17-jre-headless unzip

# AlmaLinux / Rocky / CentOS (inclusive VPS com cPanel)
dnf install -y java-17-openjdk-headless unzip
```

Confira: `java -version` deve mostrar 17 (ou mais novo).

### B.2 Criar o banco de dados

**Com cPanel:** *MySQL® Databases* → crie o banco (ex.: `conta_suba`), crie o usuário, e em *Add User to Database* marque **ALL PRIVILEGES**. Anote os nomes completos (o cPanel acrescenta um prefixo).

**Sem cPanel:**

```bash
# Ubuntu: apt install -y mariadb-server      AlmaLinux: dnf install -y mariadb-server && systemctl enable --now mariadb
mysql -u root <<'SQL'
CREATE DATABASE suba_portal CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'suba_portal'@'localhost' IDENTIFIED BY 'TROQUE-ESTA-SENHA';
GRANT ALL PRIVILEGES ON suba_portal.* TO 'suba_portal'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Para saber se o servidor é MySQL ou MariaDB: `mysql --version`.

### B.3 Copiar os arquivos

```bash
useradd --system --home /opt/suba-portal --shell /usr/sbin/nologin suba
mkdir -p /opt/suba-portal/lib /opt/suba-portal/arquivos
```

Envie o pacote para o VPS (por SFTP, FileZilla ou `scp`) e copie:

```bash
cd /caminho/onde/descompactou/portal
cp backend/suba-portal.jar /opt/suba-portal/
cp backend/config.exemplo.properties /opt/suba-portal/config.properties
cp -r web /opt/suba-portal/web
cp deploy/vps/backup.sh /opt/suba-portal/
```

### B.4 Baixar o driver do banco

O driver é o arquivo que deixa o Java falar com o banco. Baixe **o do seu banco** para a pasta `lib`, com exatamente este nome:

```bash
cd /opt/suba-portal/lib
# MySQL:
curl -L -o mysql-connector-j.jar https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
# MariaDB:
curl -L -o mariadb-java-client.jar https://repo1.maven.org/maven2/org/mariadb/jdbc/mariadb-java-client/3.4.1/mariadb-java-client-3.4.1.jar
```

Se o endereço mudar, os mesmos arquivos estão nas páginas oficiais: *MySQL Connector/J* (dev.mysql.com/downloads/connector/j, opção "Platform Independent") e *MariaDB Connector/J* (mariadb.com/downloads/connectors). Qualquer versão recente serve; só mantenha o nome do arquivo.

### B.5 Preencher a configuração

```bash
nano /opt/suba-portal/config.properties
```

Preencha pelo menos:

- `url.publica` — o endereço com https, ex.: `https://membros.seudominio.com.br`
- `db.url`, `db.usuario`, `db.senha` — para MariaDB use a linha `jdbc:mariadb://localhost:3306/NOME_DO_BANCO`
- `admin.nome`, `admin.email`, `admin.senha` — o primeiro acesso do mentor (8+ caracteres, com letras e números)
- `smtp.*` — veja a seção **4. E-mail**

Proteja o arquivo e teste:

```bash
chown -R suba:suba /opt/suba-portal
chmod 600 /opt/suba-portal/config.properties
cd /opt/suba-portal && sudo -u suba java -jar suba-portal.jar
```

Deve aparecer `Banco de dados: aplicando V001__esquema_inicial.sql`, `Primeiro acesso criado para o mentor ...` e `Área de Membros 1.0.0 no ar`. Pare com `Ctrl+C`.

Mensagens comuns nesta hora:

| Mensagem | Causa |
|---|---|
| `Driver do banco não encontrado` | o arquivo da pasta `lib` está com outro nome, ou `db.url` começa com `jdbc:mysql` e o driver baixado foi o do MariaDB (ou o contrário) |
| `Não foi possível conectar ao banco de dados` | usuário, senha ou nome do banco errados em `db.*` |
| `A pasta de arquivos não aceita gravação` | faltou o `chown -R suba:suba /opt/suba-portal` |

### B.6 Deixar ligado sempre (systemd)

```bash
cp /caminho/portal/deploy/vps/suba-portal.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now suba-portal
systemctl status suba-portal        # deve mostrar "active (running)"
journalctl -u suba-portal -f        # registro do servidor ao vivo (Ctrl+C sai)
```

### B.7 Endereço público com HTTPS

O Java escuta só dentro da máquina (porta 8080). Quem recebe o público é o Nginx ou o Apache.

**VPS sem cPanel (Nginx):**

```bash
apt install -y nginx certbot python3-certbot-nginx        # AlmaLinux: dnf install -y nginx certbot python3-certbot-nginx
cp /caminho/portal/deploy/vps/nginx.conf /etc/nginx/conf.d/suba-portal.conf
sed -i 's/membros.seudominio.com.br/SEU-ENDERECO/' /etc/nginx/conf.d/suba-portal.conf
nginx -t && systemctl enable --now nginx && systemctl reload nginx
certbot --nginx -d SEU-ENDERECO        # emite o certificado e liga o HTTPS (renova sozinho)
```

**VPS com cPanel/WHM (Apache):**

1. No cPanel, crie o subdomínio `membros.seudominio.com.br` e rode o **AutoSSL** (*SSL/TLS Status → Run AutoSSL*).
2. No WHM, abra *Service Configuration → Apache Configuration → Include Editor → Pre VirtualHost Include* **ou** crie os arquivos abaixo por SSH (troque `USUARIO` pela conta do cPanel e o domínio):

   ```bash
   for tipo in std ssl; do
     mkdir -p /etc/apache2/conf.d/userdata/$tipo/2_4/USUARIO/membros.seudominio.com.br
     cat > /etc/apache2/conf.d/userdata/$tipo/2_4/USUARIO/membros.seudominio.com.br/portal.conf <<'CONF'
   ProxyPreserveHost On
   ProxyRequests Off
   ProxyTimeout 120
   LimitRequestBody 26214400
   RequestHeader set X-Forwarded-For "%{REMOTE_ADDR}s"
   ProxyPass /.well-known !
   ProxyPass / http://127.0.0.1:8080/
   ProxyPassReverse / http://127.0.0.1:8080/
   CONF
   done
   /scripts/rebuildhttpdconf && systemctl restart httpd
   ```

Abra `https://membros.seudominio.com.br` no navegador: deve aparecer a tela de login.

Firewall: deixe abertas só as portas 22 (SSH), 80 e 443. A 8080 e a 3306 **não** devem ficar abertas para a internet.

---

## 3. Primeiro acesso

1. Entre com o `admin.email` e a `admin.senha` da configuração.
2. Em **Configurações → Acessos ao painel**, clique em *Alterar minha senha*.
3. Apague a linha `admin.senha` do `config.properties` (ou a variável `SOP_ADMIN_SENHA`). Ela só é usada quando não existe nenhum mentor.
4. Outros mentores: *Configurações → Acessos ao painel → Novo acesso*. Não existe tela pública de cadastro.
5. Alunos: *Alunos → Cadastrar aluno*. O mentor define a senha e repassa; o aluno pode trocar no perfil.

Pelo terminal do servidor (quando ninguém consegue entrar):

```bash
cd /opt/suba-portal
sudo -u suba java -jar suba-portal.jar criar-mentor "Nome" email@dominio.com SenhaForte123
sudo -u suba java -jar suba-portal.jar redefinir-senha email@dominio.com NovaSenha123
```

O portal vem com 5 competências e 10 perguntas frequentes iniciais. São editáveis (e apagáveis) em *Competências* e *Suporte*.

## 4. E-mail (avisos e "Esqueci minha senha")

Use a caixa criada para o portal. Na HostGator os dados costumam ser:

```
smtp.host=mail.seudominio.com.br
smtp.porta=465
smtp.seguranca=ssl
smtp.usuario=portal@seudominio.com.br      (o e-mail completo)
smtp.senha=senha da caixa
smtp.remetente=portal@seudominio.com.br
```

Alternativa: porta `587` com `smtp.seguranca=starttls`.

Depois de reiniciar o servidor, teste em **Configurações → Integrações → E-mail → Enviar teste**. Se falhar, a tela mostra a resposta do provedor:

- **"recusou o usuário ou a senha"** — confira `smtp.usuario` (e-mail completo) e `smtp.senha`.
- **"Não foi possível abrir conexão segura ... certificado"** — em alguns planos o certificado do servidor de e-mail está no nome do servidor da HostGator, não no `mail.seudominio`. Use em `smtp.host` o nome do servidor que aparece no cPanel em *Contas de E-mail → Connect Devices* (ex.: `br123.hostgator.com.br`).
- Para os e-mails não caírem em spam, confira no DNS do domínio os registros **SPF** e **DKIM** (no cPanel: *Email Deliverability*).

Sem SMTP configurado o portal funciona normalmente: os avisos aparecem dentro do portal e ficam em *Notificações → E-mail e WhatsApp* para envio manual com um clique.

## 5. Notificações no celular (PWA e push)

Não há nada para configurar no servidor: as chaves do push são criadas sozinhas na primeira execução. **Exige HTTPS.**

Oriente cada pessoa:

- **Android (Chrome):** abrir o portal → *Instalar aplicativo* → em *Perfil*, ligar *Notificações neste aparelho*.
- **iPhone (Safari, iOS 16.4+):** abrir o portal → botão Compartilhar → *Adicionar à Tela de Início* → abrir pelo ícone → em *Perfil*, ligar *Notificações neste aparelho*. No iPhone o push só funciona com o portal instalado.
- **Computador:** Chrome ou Edge → ícone de instalar na barra de endereço.

## 6. Google Agenda

Cada pessoa tem um endereço próprio de agenda. Em *Agenda → Assinar a agenda* ela escolhe Google Agenda, iPhone/Mac/Outlook ou copia o endereço. Os encontros e prazos da turma aparecem na agenda dela e se atualizam sozinhos quando o mentor muda uma data (o Google confere agendas assinadas algumas vezes por dia, então uma mudança pode levar horas para aparecer lá). Não é preciso criar nada no Google Cloud.

## 7. WhatsApp

O portal não envia WhatsApp sozinho (isso exige a API oficial paga da Meta ou um serviço de automação). Há dois modos:

- **Manual (padrão):** cada mensagem fica pronta em *Notificações → E-mail e WhatsApp*; um clique abre o WhatsApp com o texto.
- **Automático:** em *Notificações → Automação*, cole o endereço https de um webhook (n8n, Make, Zapier). O portal entrega ali `{canal, para, nome, assunto, mensagem}` e o seu fluxo envia.

## 8. Backup

O que guardar: **o banco** e **a pasta `arquivos`**.

```bash
mkdir -p /var/backups/suba-portal
/opt/suba-portal/backup.sh                                  # teste
echo '30 3 * * * root /opt/suba-portal/backup.sh >> /var/log/suba-backup.log 2>&1' > /etc/cron.d/suba-backup
```

O script guarda 14 dias em `/var/backups/suba-portal`. Copie essa pasta para fora do VPS de tempos em tempos. No EasyPanel, use o backup do serviço MySQL e do volume.

Restaurar: `gunzip < banco-DATA.sql.gz | mysql -u USUARIO -p NOME_DO_BANCO` e `tar -xzf arquivos-DATA.tar.gz -C /opt/suba-portal`.

Além disso, o mentor tem em *Configurações → Cópia dos dados* um arquivo com turmas, alunos, notas e frequência (sem os arquivos enviados).

## 9. Atualizar

- **Só as páginas** (pasta `web`): substitua os arquivos em `/opt/suba-portal/web`. Não precisa reiniciar.
- **O servidor** (`suba-portal.jar`): faça backup, substitua o arquivo e rode `systemctl restart suba-portal`. Mudanças de banco são aplicadas sozinhas na subida.
- **EasyPanel:** envie as mudanças ao repositório e clique em *Deploy*.

Gerar o `.jar` a partir do código: `cd backend && ./compilar.sh` (só precisa do JDK 17) ou `mvn package` (o `.jar` do Maven sai em `target/` já com os drivers dentro).

## 10. Conferência final

- [ ] `https://SEU-ENDERECO/api/saude` responde `{"ok":true,...}`
- [ ] Login do mentor funciona e a senha inicial foi trocada
- [ ] `admin.senha` apagada da configuração
- [ ] E-mail de teste chegou
- [ ] Um aluno de teste foi cadastrado, entrou, e depois foi removido (*ficha do aluno → Dados → Excluir dados*)
- [ ] Aplicativo instalado no celular e notificação de teste recebida
- [ ] Backup agendado e testado
- [ ] Portas 8080 e 3306 fechadas para a internet
