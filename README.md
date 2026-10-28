# Contatos Rebeldia

Site para acompanhar os inscritos da Rebeldia. Ele substitui a planilha "inscrições Rebeldia Total".

Endereço: https://deniormachado.github.io/contatos-rebeldia/

## Como funciona
- O site é um único arquivo (`index.html`), publicado pelo GitHub Pages.
- Os dados ficam no Supabase, não neste repositório.
- Só entra quem faz login pelo link enviado por e-mail **e** está na lista de usuários autorizados. Sem login, nenhum dado aparece.

## Como dar acesso a uma pessoa
- No site, o administrador clica no botão **Usuários** e inclui o e-mail e o nome da pessoa.
- Outra forma: incluir o e-mail (em minúsculas) na tabela `usuarios_autorizados` do Supabase.

Dica: cadastre o nome igual ao usado no campo "Responsável" (ex.: "Ícaro"), para o filtro "Meus contatos" funcionar.
