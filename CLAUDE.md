# Contatos Rebeldia — contexto do projeto

## Objetivo
Substituir a planilha Google "inscrições Rebeldia Total" por um sistema simples. Os dados ficam no Supabase e são usados por um site estático publicado no GitHub Pages. Esse site oferece:
- uma visão em grade, como uma planilha, com busca, filtros e ordenação;
- um formulário que inclui ou edita a linha inteira de uma vez;
- acesso restrito a algumas pessoas autorizadas.

Quem usa: militantes que fazem o contato com os inscritos. São usuários não técnicos, muitas vezes no celular. O dono e administrador é Denior (denior.machado@gmail.com).

## Diretrizes
- **Simplicidade acima de tudo.** O site é um único `index.html`, sem etapa de build e sem framework. HTML, CSS e JS ficam no mesmo arquivo. Só mude essa estrutura se o usuário pedir.
- **Interface em português do Brasil**, com linguagem simples e sem jargão técnico.
- **Funcionar bem no celular.** A página não pode rolar na horizontal; só a tabela rola, dentro da própria caixa.
- **São dados pessoais.** Entre eles há menores de idade, telefones e comentários. A segurança fica no banco (RLS), nunca só no front-end.
- **Nunca** coloque no repositório a chave `service_role` / `sb_secret_`, senhas ou credenciais de SMTP. A chave `sb_publishable_` do `index.html` é pública por natureza.
- Mudanças no banco devem ser feitas por migração (SQL versionado). Antes de alterar o esquema, confira a estrutura atual.
- Antes de qualquer alteração em massa nos dados (atualizar ou apagar muitas linhas), mostre ao usuário o que vai mudar e peça confirmação.

## Supabase
- Projeto: `contatos-rebeldia`. Ref: `singuutlpjdpfogdprip`. Região: sa-east-1. Plano gratuito.
- URL: `https://singuutlpjdpfogdprip.supabase.co`

### Tabela `public.contatos`
| coluna | tipo | observação |
|---|---|---|
| id | bigint identity | chave |
| regiao | text | Central, Litoral, Metropolitana, Noroeste, Oeste, Planalto Médio, S. Cruz do Sul, Serra, Sul, Vale do Taquari |
| nome | text not null | |
| instagram | text | formato `@usuario` |
| telefone | text | padrão `(DD) 9XXXX-XXXX` |
| telefone_conferir | boolean | marca telefones com dígitos faltando ou sobrando |
| idade | int | 0–120 |
| local | text | cidade / bairro |
| estuda_trabalha | text | |
| comentario_inscricao | text | texto original do formulário de inscrição |
| observacoes | text | |
| email | text | minúsculo |
| origem | text | Faça Parte (com variações do site), Rebeldia (impulsionamento), Apresentação Litoral-Serra |
| situacao | text | check: Simpatizante / Filiado / Filiado e contribuinte |
| data_cadastro | date | data vinda do formulário "Faça Parte" |
| responsavel | text | quem faz o contato (ex.: Ícaro, Denior) |
| retorno_contato | text | |
| caracterizacao | text | |
| data_inclusao | date | automática (dia em que entrou no sistema, fuso de São Paulo); não muda na edição |
| status_acomp | text not null | check: A contatar / Em andamento / Sem resposta / Concluído (padrão A contatar) |
| ultimo_contato | date | data do registro de acompanhamento mais recente (via trigger) |

Observação: `responsavel`, `retorno_contato` e `caracterizacao` funcionam como **resumo**. Eles mostram o registro de acompanhamento mais recente e são atualizados pela trigger `contatos_resumir_acompanhamento`. O histórico completo fica em `acompanhamentos`.

### Tabela `public.acompanhamentos` (histórico)
Colunas:
- `id`;
- `contato_id`, que aponta para `contatos` com `on delete cascade`;
- `data`, com padrão no dia de hoje (fuso de São Paulo);
- `responsavel`, `retorno`, `caracterizacao` e `observacoes`; pelo menos um entre retorno, caracterização e observações é obrigatório;
- `criado_em` e `criado_por`.

A trigger `acomp_resumo` roda depois de inserir, alterar ou excluir um registro e atualiza a ficha do contato:
- `ultimo_contato` recebe a data do registro mais recente;
- `retorno_contato`, `caracterizacao` e `responsavel` recebem os valores desse registro, quando ele os tiver;
- num novo registro, `status_acomp` passa de "A contatar" para "Em andamento". Os demais status são mudados à mão.
| criado_em / criado_por | timestamptz / text | automáticos |
| atualizado_em / atualizado_por | timestamptz / text | preenchidos pela trigger `contatos_marcar_alteracao` |

### Tabela `public.usuarios_autorizados`
Colunas: `email` (pk, minúsculo), `nome`, `admin` (boolean) e `criado_em`. Denior está cadastrado como admin.

### Segurança (RLS)
- Em `contatos`, as quatro operações (ler, incluir, alterar, excluir) só são permitidas a quem está logado e passa em `privado.eh_autorizado()`, ou seja, cujo e-mail do JWT está em `usuarios_autorizados`.
- Em `usuarios_autorizados`, cada usuário vê só o próprio registro. O admin (`privado.eh_admin()`) vê todos, inclui, altera e exclui, mas não pode excluir a si mesmo.
- Em `acompanhamentos`, qualquer autorizado lê e registra. Só o autor do registro (`criado_por`) ou o admin podem alterar ou excluir.
- O papel `anon` não tem nenhum privilégio nas tabelas.
- As funções `security definer` ficam no schema `privado`, fora da API, para não gerar aviso no verificador de segurança do Supabase.
- Depois de mudar o esquema, rode o verificador de segurança do Supabase (advisors) e mantenha o resultado sem avisos.

### Login
- É por *magic link* (`signInWithOtp`), com redirecionamento para o próprio site.
- Requer a Site URL e as Redirect URLs configuradas para o endereço do GitHub Pages.
- Requer SMTP próprio. O e-mail padrão do Supabase só entrega a membros da equipe do projeto.

## Site (`index.html`)
- Usa `@supabase/supabase-js@2.45.4` via jsDelivr (UMD, `dist/umd/supabase.js`).
- Telas:
  - login;
  - "sem autorização";
  - app, com estas partes:
    - cartões de resumo que também servem de filtro: Total, Meus contatos, A contatar, Em andamento, Sem resposta, Concluído, Sem responsável, Parados (+7 dias sem registro), Telefone a conferir, Menores de 18;
    - busca sem distinção de acentos, que também procura no histórico de acompanhamento;
    - filtros por região, origem, situação (Faça Parte) e responsável;
    - tabela ordenável, com o selo de acompanhamento colorido; clicar no selo abre direto a aba Acompanhamento;
    - telefone com link para o WhatsApp e Instagram com link para o perfil;
    - ficha em `<dialog>` com duas abas:
      - **Dados**: cadastro, com inclusão, alteração e exclusão confirmada, formatação automática do telefone e aviso de duplicata;
      - **Acompanhamento**: responsável e situação atuais; formulário "Registrar contato" (data, quem fez, retorno, caracterização, observações); histórico do mais novo para o mais antigo, com Editar e Excluir para o autor ou o admin; anotações antigas da planilha aparecem como "Anotação trazida da planilha";
    - exportação de CSV (separador `;`, com BOM, para abrir no Excel);
    - painel "Usuários", só para admin, para dar e tirar acesso.
- "Meus contatos" compara o campo `responsavel` com `usuarios_autorizados.nome` do usuário logado. Por isso o nome cadastrado deve ser igual ao usado em Responsável (ex.: "Ícaro").
- Endereço: `https://deniormachado.github.io/contatos-rebeldia/`

## Histórico da migração (28/10/2026)
- 72 linhas da planilha viraram 70 contatos. Foram removidas duas duplicatas: Orimar Luis Pizzamiglio e Osmar Marques.
- "Metropilitana" foi unificada como "Metropolitana".
- Os telefones foram padronizados. Seis ficaram com `telefone_conferir = true`: Alessandra Colares, Artemis Santos, Vera Beatriz Gonzaga, Ana Luisa Rodrigues Borges, João Victor Brum de Carvalho e Victor Luiz Do Carmo Bretanha.
- Os textos "(não tem)" e "(banido do Instagram no momento)" saíram da coluna Instagram e foram para Observações.
- Na linha 18 (Andrey Assis), "Já tá militando" foi movido de Responsável para Caracterização.
- A aba "Resumo" da planilha tinha #REF! e uma contagem errada de menores. O site calcula o resumo sozinho.
- Os 70 contatos importados estão com `data_inclusao = 2026-10-28`.
- Quando o acompanhamento foi criado, os 9 contatos que já tinham responsável ou caracterização receberam o status "Em andamento". Os outros 61 ficaram "A contatar".

## Pendências
- [ ] Configurar SMTP próprio no Supabase (Gmail com senha de app ou Resend gratuito).
- [ ] Incluir os e-mails dos outros usuários autorizados (ex.: Ícaro), com o nome igual ao usado em Responsável.
- [ ] Decidir se, nos contatos importados que têm `data_cadastro`, a `data_inclusao` deve receber essa data.
- [ ] Opcional: modelo do e-mail de login em português.
