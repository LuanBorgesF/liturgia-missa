# Cantos & Liturgia — Livro de Cantos

Aplicativo de página única para o repertório de cantos da missa: biblioteca
com cifras e transposição de tom, montagem da missa por momento litúrgico,
salmo responsorial e aclamação ao Evangelho do dia, e geração de PDF para
impressão.

Hoje roda como Artifact no Claude. Este repositório é o código-fonte,
organizado para poder ser hospedado em servidor comum.

## Estrutura

    index.html      marcação da página (as 4 abas e os diálogos)
    styles.css      todo o visual, incluindo tema claro/escuro
    app.js          toda a lógica (~2.300 linhas, JavaScript puro, sem framework)
    assets/         imagens (o traço da catedral usado como fundo)
    dados/          export do conteúdo atual do banco (ver abaixo)
    build.py        gera dist/livro-de-cantos.html, a versão de arquivo único
    dist/           saída do build.py (fora do versionamento)

Sem dependências de build, sem `npm install`. A única biblioteca externa é a
jsPDF, carregada sob demanda por CDN só quando se gera um PDF.

## Rodar localmente

    cd cantos-liturgia
    python -m http.server 8000

E abrir <http://localhost:8000>. Abrir o `index.html` direto pelo disco
(`file://`) também mostra a página, mas alguns navegadores bloqueiam parte
dos recursos — o servidor local evita esse tipo de surpresa.

A página vai carregar e mostrar o aviso de banco indisponível. Isso é
esperado: veja a seção seguinte.

## O ponto que precisa de trabalho para hospedar

O aplicativo não tem backend próprio. Ele conversa com o banco de dados que
o runtime do Claude oferece à página publicada, obtido em `app.js`:

    // app.js, dentro de boot(), por volta da linha 2211
    window.claude.use("db")

Fora do Claude essa chamada não existe, `db` fica `null` e a biblioteca
aparece vazia com a mensagem "Não foi possível conectar ao banco de dados".
**Hospedar a pasta num servidor, sozinho, não faz o app funcionar.** É
preciso fornecer um objeto `db` equivalente.

A boa notícia é que a interface usada é a do Firestore, e apenas um
subconjunto dela:

    db.collection(nome)                  -> referência de coleção
    db.collection(nome).doc(id)          -> referência de documento
      .get()                             leitura avulsa
      .set(obj)                          grava o documento inteiro
      .update(obj)                       mescla campos (exige documento existente)
      .delete()
    db.collection(nome).add(obj)         cria com id gerado
    db.collection(nome).onSnapshot(cb)   assina a coleção, em tempo real

As coleções usadas são `songs`, `missas`, `rascunhos`, `salmos`,
`aclamacoes`, `pedidos` e `config`.

Caminhos possíveis, do mais curto ao mais longo:

1. **Firebase / Firestore.** A API bate quase 1:1 com o que o código já
   chama — na prática é trocar a linha do `window.claude.use("db")` pela
   inicialização do Firestore e manter todo o resto. Tem plano gratuito e
   resolve autenticação e regras de acesso por usuário, que é justamente o
   que hoje impede duas pessoas de editarem juntas.
2. **Supabase.** Banco Postgres com API pronta; exige escrever uma camada
   fina que traduza as chamadas acima para as do cliente do Supabase.
3. **Backend próprio** (Node, .NET, o que for) com um banco qualquer e a
   mesma camada de tradução. Mais controle, mais trabalho, e passa a exigir
   um servidor de verdade em vez de hospedagem estática.

Enquanto isso não existir, dá para deixar o app utilizável em modo
demonstração implementando esse mesmo objeto sobre `localStorage`: funciona
offline e por navegador, sem compartilhar nada entre pessoas.

## dados/

Export do conteúdo do banco no momento em que este repositório foi gerado
(17/09/2026). Cada arquivo é um array de documentos, com o `id` de cada um:

    songs.json        432 cantos (título, categoria, tom, ritmo, cifra)
    missas.json       1 missa montada
    salmos.json       99 salmos responsoriais, de 16/09/2026 a 01/01/2027
    aclamacoes.json   81 aclamações ao Evangelho
    config.json       controle da rotina que busca salmo e aclamação

Serve para dois fins: é a cópia de segurança do repertório e é a carga
inicial de qualquer banco novo que venha a substituir o do Claude.

Os salmos e as aclamações vêm da API de liturgia diária
(`api-liturgia-diaria.vercel.app`, com `liturgia.up.railway.app` como
reserva), preenchidos por uma tarefa agendada. Os cantos foram cadastrados
pelo grupo, dentro do próprio app.

## Voltar para o Artifact

Alterou o código e quer atualizar a versão publicada no Claude:

    python build.py

O arquivo `dist/livro-de-cantos.html` sai autocontido — CSS, JavaScript e
imagens embutidos — que é o formato exigido lá. A pasta solta continua
sendo a fonte de verdade; `dist/` é só o resultado.

## Licença e conteúdo

O código é seu. As letras e cifras cadastradas pertencem a seus autores e
editoras — publicar o repertório abertamente na internet é uma decisão
diferente de hospedar a ferramenta, e vale verificar antes de tornar o site
público.
