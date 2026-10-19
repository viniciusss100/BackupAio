# Tarefa: integrar AIOStreams, bibliotecas virtuais e Plex no VPS

Você é um engenheiro sênior de Linux, Docker Compose, bibliotecas de mídia, WebDAV, Plex e integração de aplicações. Trabalhe diretamente no servidor e no repositório/configuração existentes, investigando primeiro a arquitetura real antes de alterar arquivos.

## Objetivo

Configurar uma solução na qual:

1. O AIOStreams continue funcionando como já está configurado.
2. Seja criada uma biblioteca de mídia que o Plex consiga indexar, formada por arquivos `.strm` e/ou symlinks quando tecnicamente apropriado, apontando para conteúdos acessíveis por WebDAV.
3. Seja possível criar bibliotecas de filmes e séries a partir dos catálogos de addons externos integrados ao AIOStreams, desde que os addons ofereçam os metadados e identificadores necessários e que a integração seja tecnicamente viável.
4. Seja instalada e configurada uma instância do Plex Media Server no mesmo servidor, preferencialmente em Docker Compose, para ler e indexar essas bibliotecas.
5. A solução preserve os dados, credenciais, redes, volumes, proxy reverso e demais serviços existentes.

## Contexto conhecido — confirme no servidor antes de agir

- Servidor VPS com Ubuntu 24 e Docker/Compose.
- AIOStreams foi implantado em `/home/ubuntu/docker/aiostreams` ou em `~/docker/aiostreams`; descubra o caminho real, não presuma.
- AIOStreams usa ou usou a imagem `ghcr.io/viren070/aiostreams:latest`, porta interna `3054`, dados persistentes e a rede Docker externa `proxy`.
- O acesso externo pode passar pelo Nginx Proxy Manager (NPM). No VPS, não configure VPN para o AIOStreams.
- Há ou houve outros serviços Docker no servidor. Não altere suas configurações por conveniência.

Os dados acima são pistas de contexto, não substituem a inspeção da máquina. Determine o estado atual antes de editar.

## Fase 1 — inspeção obrigatória, sem alterações destrutivas

1. Identifique o usuário, sistema operacional, arquitetura, espaço em disco e versões de Docker e Docker Compose.
2. Inspecione os diretórios de projeto, arquivos Compose, `.env` (verifique apenas a presença das variáveis; nunca imprima ou copie segredos para logs ou relatórios), redes, volumes e containers existentes.
3. Localize a configuração efetiva do AIOStreams e identifique a imagem/tag em uso, diretório de dados, porta interna, URL base e rede Docker.
4. Procure configurações e mounts existentes de WebDAV, rclone, RaiDrive, Rclone Union ou outros provedores de armazenamento. Identifique caminhos montados, opções de montagem, permissões e se o WebDAV contém arquivos reais ou streams remotos.
5. Identifique o proxy reverso existente e como o Plex poderá ser acessado. Não exponha portas públicas desnecessariamente.
6. Leia a documentação oficial atual do AIOStreams, do protocolo de addons Stremio, do Plex e das ferramentas que forem consideradas. Não invente endpoints, variáveis de ambiente, APIs ou funcionalidades.
7. Antes de modificar qualquer arquivo, apresente um resumo do que encontrou, a arquitetura proposta, riscos e arquivos que pretende alterar. Se o ambiente estiver em modo de execução autônoma, prossiga apenas com ações reversíveis e seguras; peça aprovação antes de mudanças destrutivas, abertura de portas públicas, alterações de firewall ou reinicializações que possam interromper serviços importantes.

## Fase 2 — definir como transformar as fontes em biblioteca

Avalie separadamente as duas fontes:

### A. Conteúdo disponível via WebDAV

- Verifique quais mounts WebDAV existem e como são acessados.
- Prefira `.strm` quando a origem oferece URLs de reprodução HTTP(S) utilizáveis pelo Plex. O arquivo deve conter a URL direta de reprodução e não uma página HTML, URL de catálogo ou endpoint que exija autenticação interativa.
- Use symlinks apenas quando o conteúdo estiver acessível como arquivos locais/montados no sistema de arquivos e os symlinks forem válidos dentro dos volumes visíveis pelo Plex.
- Não presuma que o Plex conseguirá ler uma URL WebDAV autenticada só porque o cliente WebDAV consegue listar arquivos. Teste as permissões, autenticação, alcance de rede e suporte real à reprodução.
- Se a origem exigir autenticação HTTP que o Plex não consegue usar para reproduzir a URL gravada no `.strm`, não grave credenciais no arquivo nem em logs. Avalie um mecanismo compatível e seguro, como um mount que apresente os arquivos ao sistema de arquivos, ou um proxy local autenticado que não exponha credenciais ao cliente. Documente os trade-offs e não implemente proxy inseguro.
- Não baixe arquivos de vídeo inteiros para o disco local como parte da indexação. A solução deve ser virtual/streaming, salvo se eu autorizar explicitamente o contrário.
- Considere que symlinks podem apontar para destinos inexistentes dentro do container. Monte os diretórios e defina UID/GID e permissões corretamente.

### B. Catálogos de addons externos integrados ao AIOStreams

Primeiro determine o que o AIOStreams realmente disponibiliza para outros sistemas e o que cada addon oferece. O protocolo Stremio normalmente fornece metadados de catálogo e streams, mas isso não equivale a uma biblioteca Plex nativa nem garante URLs permanentes de reprodução.

- Verifique se é possível obter catálogos, metadados, IDs IMDb/TMDB ou outros IDs, temporadas, episódios, nomes, ano, poster e URLs de streams por meios oficialmente suportados.
- Não assuma que os catálogos do AIOStreams podem ser exportados diretamente para o Plex. Confirme a disponibilidade de endpoints e a compatibilidade real da API.
- Para filmes e séries, gere `.strm` apenas se houver uma URL de stream válida, acessível ao Plex e compatível com seu mecanismo de reprodução. Considere que URLs de stream podem ser temporárias, assinadas, exigir headers, depender de sessão ou variar a cada consulta.
- Não trate URLs de busca, páginas de detalhes, manifests de addons ou IDs de conteúdo como se fossem URLs de vídeo.
- Se for necessário um serviço intermediário, implemente-o como um serviço separado, com responsabilidades explícitas: sincronizar catálogo, manter metadados, resolver stream sob demanda e servir `.strm` ou redirecionar para uma URL válida, somente se os protocolos e termos do serviço permitirem.
- Não tente contornar DRM, autenticação, paywalls ou restrições de acesso. Use apenas conteúdos e serviços aos quais o usuário tem acesso legítimo.
- Para séries, crie a estrutura esperada pelo Plex, por exemplo `Series/Nome da Série (Ano)/Season 01/Nome da Série - S01E01.strm`, com metadados e IDs quando disponíveis. Para filmes, use `Movies/Nome do Filme (Ano)/Nome do Filme (Ano).strm`.
- Use IDs estáveis para evitar duplicatas e preservar correspondência com metadados. Trate episódios especiais, temporadas desconhecidas, títulos em português e conteúdo sem ano de forma segura.
- Se a sincronização de catálogo não puder ser feita de forma confiável por APIs oficiais, não raspe interfaces privadas sem autorização. Relate exatamente o que é e não é possível.

## Fase 3 — arquitetura e persistência

Implemente uma estrutura organizada, por exemplo, adaptada aos caminhos reais:

- `.../plex/config` — configuração persistente do Plex.
- `.../media/Movies` — biblioteca de filmes.
- `.../media/TV Shows` — biblioteca de séries.
- `.../media/strm-webdav` — `.strm` gerados para a fonte WebDAV, se necessário.
- `.../media/strm-addons` — `.strm` gerados para addons, se a resolução de streams for viável.
- `.../catalog-sync` — configuração, estado e logs do sincronizador.

A estrutura acima é sugestão, não crie diretórios duplicados se já houver estrutura equivalente. Separe os arquivos gerados dos dados originais. Nunca misture a pasta de configuração do Plex com as bibliotecas de mídia.

No Compose:

- Crie o Plex como serviço próprio, com versão/tag de imagem explicitamente definida e compatível com a arquitetura do VPS.
- Use volumes persistentes para configuração e bibliotecas. Não use `privileged: true` sem necessidade demonstrada.
- Conecte o Plex às redes necessárias. Reutilize a rede externa `proxy` quando fizer sentido, sem anexar serviços a redes desnecessárias.
- Evite conflitos de portas. Se o NPM for usado, configure o acesso por ele de acordo com os requisitos do Plex e explique eventuais limitações de proxy/HTTPS. Não altere hosts existentes no NPM sem aprovação.
- Não publique a interface administrativa ou portas de streaming na internet sem necessidade. Prefira acesso privado/local, VPN existente do usuário ou regras explícitas de firewall. Não altere firewall automaticamente.
- Configure UID/GID, permissões, healthcheck quando suportado, restart policy e logs rotacionados.
- Garanta que o container do Plex enxergue os caminhos corretos de arquivos e destinos de symlinks.
- Mantenha credenciais em `.env` ou mecanismo seguro já usado no projeto; não comite segredos.
- Não altere nem regenere secrets existentes do AIOStreams.
- Não remova containers, volumes, imagens, redes ou arquivos que não pertençam diretamente à implementação aprovada.

## Fase 4 — sincronizador de biblioteca

Se for necessário implementar um sincronizador, faça-o como um serviço isolado, com código legível e documentação.

Requisitos mínimos:

1. Sincronização incremental e idempotente: executar repetidamente não deve criar duplicatas.
2. Persistir IDs, metadados e relação entre entradas do catálogo e arquivos gerados.
3. Não apagar automaticamente arquivos que falharam numa única consulta à API. Use uma política de expiração e uma janela de falhas configurável.
4. Tratar rate limits, timeouts, erros de rede e respostas incompletas.
5. Não registrar URLs assinadas, tokens, senhas, cookies ou headers sensíveis.
6. Não baixar o conteúdo dos vídeos; gerar apenas arquivos de referência e metadados necessários.
7. Separar filmes e séries e nomear arquivos conforme convenções reconhecidas pelo Plex.
8. Oferecer modo `dry-run`, relatório de adições/alterações/remoções e logs úteis.
9. Se houver sincronização periódica, usar intervalo configurável e respeitar limites do provedor.
10. Manter os arquivos `.strm` atualizados caso as URLs expirem, se houver forma autorizada e suportada de resolver os streams novamente.
11. Não inventar disponibilidade de episódios nem criar entradas sem um stream reproduzível.
12. Incluir um README com instalação, configuração, atualização, operação, limitações e recuperação.

Não construa o sincronizador antes de provar, com um teste pequeno, que é possível obter metadados e URLs de stream utilizáveis de cada fonte. Comece por um filme e um episódio de teste.

## Fase 5 — testes de aceitação

Execute e registre os resultados, sem revelar segredos:

- `docker compose config` passa para os arquivos alterados.
- AIOStreams continua saudável e acessível pela URL atual.
- O Plex inicia e mantém sua configuração após reinicialização.
- O Plex enxerga as pastas de biblioteca e consegue ler os `.strm` ou symlinks de teste.
- Um item da fonte WebDAV é listado e, quando a fonte suportar, reproduzido de ponta a ponta.
- Um filme e um episódio originados de addons são testados somente se a API e a URL de stream forem compatíveis.
- A sincronização repetida não cria duplicatas.
- Falha de API não apaga a biblioteca inteira.
- Nenhuma credencial é exposta nos logs ou arquivos `.strm`.
- Os demais containers, redes e rotas do NPM continuam funcionando.
- A documentação descreve claramente o que foi validado e o que não foi possível validar.

Se não houver um teste real de reprodução disponível, declare isso claramente. Não diga que funciona só porque o container iniciou ou porque o Plex indexou o nome do arquivo.

## Regras de segurança e limites

- Faça backup dos arquivos de configuração que pretende alterar e preserve cópias com data.
- Não execute `docker system prune`, não remova volumes e não apague dados para liberar espaço.
- Não sobrescreva `.env` existentes nem mostre seus valores.
- Não altere credenciais, domínios, redes, portas públicas ou configurações de proxy existentes sem necessidade comprovada e aprovação.
- Não desative autenticação ou TLS para simplificar a integração.
- Prefira alterações pequenas, reversíveis e documentadas.
- Se a arquitetura pretendida não for possível com o AIOStreams ou com os addons disponíveis, explique a limitação técnica com evidências e proponha a alternativa mínima viável. Não simule uma integração que não funciona.

## Entrega final

Ao concluir, informe:

1. Arquivos criados ou alterados e seus caminhos.
2. Arquitetura final e como o Plex recebe cada tipo de conteúdo.
3. Comandos exatos para iniciar, parar, atualizar e consultar logs.
4. Como adicionar ou retirar fontes WebDAV e addons.
5. Endereço de acesso ao Plex e eventuais configurações manuais necessárias.
6. Testes executados e resultados reais.
7. Limitações conhecidas, principalmente URLs temporárias, autenticação WebDAV e compatibilidade dos streams dos addons.
8. Backups criados e como restaurar a configuração anterior.

Trabalhe no servidor real com cautela. Inspecione antes de editar, prove a viabilidade antes de construir, e nunca afirme sucesso sem validar.
