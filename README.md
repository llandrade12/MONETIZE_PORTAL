O que vem no pacote

Back-end: Java 17 puro, com o suba-portal.jar já compilado, o código-fonte, o compilar.sh e o pom.xml.

Banco: MySQL ou MariaDB. As tabelas são criadas sozinhas na primeira subida; o script fica em backend/src/main/resources/db/.

Portal zerado: sem turmas, alunos, aulas ou tarefas de exemplo. O login temporário de admin saiu; o primeiro mentor vem do admin.* da configuração.

Recursos: notificações dentro do portal, e-mail por SMTP, push no celular, aplicativo instalável (PWA), assinatura da agenda no Google Agenda, lembretes automáticos, "Esqueci minha senha", cópia e restauração dos dados e exclusão pela LGPD.

Implantação: arquivos prontos para VPS com EasyPanel/Docker e para VPS com SSH/cPanel (serviço, Nginx, Apache e backup).

O que foi testado:

Servidor: 99 testes automáticos passando, cobrindo login, permissões, arquivos, e-mail (inclusive SSL e STARTTLS), push, agenda, lembretes, zerar/restaurar e LGPD.

Telas: o roteiro de navegador do aluno passou contra o jar final. O roteiro do mentor passou antes dos últimos ajustes e não foi repetido depois.

O que não foi testado:

MySQL/MariaDB real: os testes usaram um banco em memória em modo de compatibilidade com MySQL. O SQL é simples, mas a primeira subida no VPS é o teste de verdade.

pom.xml: este ambiente não acessa o Maven Central, então o mvn package nunca rodou. O compilar.sh foi o que gerou o jar entregue.

Arquivos de implantação: Dockerfile, docker-compose, Nginx, Apache, serviço e script de backup foram escritos mas não executados.

Versões dos drivers: não consegui confirmar os endereços de download (MySQL 8.4.0 e MariaDB 3.4.1); o tutorial indica as páginas oficiais como alternativa.

Push em aparelhos reais: a criptografia confere com o exemplo oficial do padrão, mas a entrega pelo Google/Apple só dá para ver em produção.

Telas mais novas: Configurações (acessos ao painel, zerar, restaurar) e "Esqueci minha senha" foram testadas só pela API, não clicando na tela.

O que você precisa saber:

Hospedagem: Java não roda em hospedagem compartilhada da HostGator; precisa ser VPS. O tutorial começa mostrando como descobrir o plano com o seu gestor.

WhatsApp: não sai sozinho. Fica pronto para envio com um clique, ou automático se vocês ligarem um webhook (n8n, Make, Zapier).

Google Agenda: o Google atualiza agendas assinadas algumas vezes por dia, então uma mudança de data pode levar horas para aparecer lá.

Certificado: a geração em PDF continua sendo pela impressão do navegador.

Conteúdo inicial: deixei as 5 competências e as 10 perguntas frequentes; são editáveis e apagáveis no painel. Se quiser que nasça sem elas, eu tiro.
