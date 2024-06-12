---
id: docs_workspaces
guide: docs_workspaces
layout: guide
---

{% include vars.html %}

Os Workspaces são uma nova forma de configurar a arquitetura do seu pacote, disponível por padrão a partir do Yarn 1.0. Eles permitem que você configure múltiplos pacotes de modo que você precise executar `yarn install` apenas uma vez para instalar todos eles.

### Por que você usaria isso? <a class="toc" id="toc-why-would-you-want-to-do-this" href="#toc-why-would-you-want-to-do-this"></a>

- Suas dependências podem ser vinculadas entre si, o que significa que seus espaços de trabalho podem depender uns dos outros, sempre utilizando o código mais atualizado disponível. Este é um mecanismo melhor que o `yarn link`, pois afeta apenas a árvore do seu workspace, em vez de todo o seu sistema.

- Todas as dependências do seu projeto serão instaladas juntas, dando ao Yarn mais liberdade para otimizá-las da melhor forma.

- O Yarn usará um único arquivo de bloqueio (lockfile) em vez de um diferente para cada projeto, resultando em menos conflitos e revisões mais fáceis.

### Como usar? <a class="toc" id="toc-how-to-use-it" href="#toc-how-to-use-it"></a>

Adicione o seguinte no arquivo `package.json`. A partir de agora, chamaremos este diretório de "raiz do workspace":

**package.json**

```json
{
  "private": true,
  "workspaces": ["workspace-a", "workspace-b"]
}
```

Note que `private: true` é obrigatório! Os workspaces não devem ser publicados, então adicionamos essa medida de segurança para garantir que nada possa expô-los acidentalmente.

Após a criação deste arquivo, crie duas novas subpastas com os nomes `workspace-a` e `workspace-b`. Em cada uma delas, crie outro arquivo `package.json` com o seguinte conteúdo:

**workspace-a/package.json:**

```json
{
  "name": "workspace-a",
  "version": "1.0.0",

  "dependencies": {
    "cross-env": "5.0.5"
  }
}
```

**workspace-b/package.json:**

```json
{
  "name": "workspace-b",
  "version": "1.0.0",

  "dependencies": {
    "cross-env": "5.0.5",
    "workspace-a": "1.0.0"
  }
}
```

Finalmente, execute `yarn install` em algum lugar, de preferência na raiz do espaço de trabalho. Se tudo funcionar bem, você deve ter uma hierarquia de arquivos semelhante a esta:

```
/package.json
/yarn.lock

/node_modules
/node_modules/cross-env
/node_modules/workspace-a -> /workspace-a

/workspace-a/package.json
/workspace-b/package.json
```

E é isso! ! Chamar o `workspace-a` de um arquivo localizado em `workspace-b` agora usará o código exato atualmente localizado em seu projeto, em vez do que é publicado no npm, e o pacote `cross-env` foi corretamente deduplicado e colocado na raiz do seu projeto para ser usado tanto por `workspace-a` quanto por `workspace-b`.

Observe que `/workspace-a` é alias para `/node_modules/workspace-a` via um symlink.
Esse é o truque que permite que você chame o pacote como se fosse um pacote comum!
Você também deve saber que o campo `/workspace-a/package.json#name` é o utilizado e não o nome da pasta.
Isso significa que se o campo `name` no `/workspace-a/package.json`  for `"pkg-a"`, o alias será o seguinte:
`/node_modules/pkg-a -> /workspace-a` e você poderá importar o código do `/workspace-a` com `const pkgA = require("pkg-a");` (ou talvez `import pkgA from "pkg-a";`).

### Como isso se compara ao Lerna? <a class="toc" id="toc-how-does-it-compare-to-lerna" href="#toc-how-does-it-compare-to-lerna"></a>

Os Workspaces do Yarn são os elementos primitivos de baixo nível que ferramentas como o Lerna podem usar (e [usam](https://github.com/lerna/lerna/pull/899)!). Eles nunca tentarão suportar os recursos de alto nível que o Lerna oferece, mas ao implementar a lógica central dos passos de resolução e vinculação dentro do próprio Yarn, esperamos possibilitar novos usos e melhorar o desempenho.

### Dicas e Truques <a class="toc" id="toc-tips-tricks" href="#toc-tips-tricks"></a>

- O campo `workspaces` é um array que contem os caminhos relativos a cada workspace. Como pode se tornar tedioso ter que localizar cada um deles, este campo também aceita os padrões glob (glob patterns) (https://www.malikbrowne.com/blog/a-beginners-guide-glob-patterns/)! Por exemplo, o Babel referencia todos os seus pacotes através de uma única diretiva `packages/*`.

- Workspaces são estáveis o suficiente para serem utilizados em aplicações de larga escala e não devem mudar nada na forma como as instalações regulares funcionam,, mas se você achar que eles estão quebrando algo, pode desativá-los adicionando a seguinte linha no seu arquivo Yarnrc:

  ```
  workspaces-experimental false
  ```

- Se você estiver fazendo alterações em um único workspace, use [--focus](/blog/2018/05/18/focused-workspaces) para instalar rapidamente dependências irmãs do registro em vez de construí-las todas do zero.

### Limitações e Advertências <a class="toc" id="toc-limitations-caveats" href="#toc-limitations-caveats"></a>

- O layout do pacote será diferente entre o seu workspace e o que seus usuários obterão (as dependências do workspace serão levantadas mais alto na hierarquia do sistema de arquivos). azer suposições sobre esse layout já era arriscado, pois o processo de levantamento não é padronizado, então teoricamente nada novo aqui. Se você encontrar problemas, tente usar [a opção `nohoist`](/blog/2018/02/15/nohoist/)

- No exemplo acima, se `workspace-b` depender de uma versão diferente da referenciada no package.json do `workspace-a`, a dependência será instalada a partir do npm em vez de ser vinculada do seu sistema de arquivos local. Isso ocorre porque alguns pacotes realmente precisam usar as versões anteriores para construir as novas (o Babel é um deles).

- Tenha cuidado ao publicar pacotes em um workspace. Se você estiver preparando sua próxima versão e decidiu usar uma nova dependência, mas esqueceu de declará-la no arquivo  `package.json`, seus testes ainda podem passar localmente se outro pacote já tiver baixado essa dependência na raiz do espaço de trabalho. No entanto, será quebrado para consumidores que o puxarem de um registro, pois a lista de dependências agora está incompleta, então eles não têm como baixar a nova dependência. Atualmente, não há como gerar um aviso neste cenário.

- Workspaces devem ser descendentes da raiz do workspace em termos de hierarquia de pastas. Você não pode e não deve referenciar um espaço de trabalho que está localizado fora dessa hierarquia de sistema de arquivos.

- Workspaces aninhados não são suportados no momento.
