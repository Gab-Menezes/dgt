# TypeStates e Builder Patterns

Builder Pattern um padrão de desenvolvimento muito usado na criação de software. Este `pattern` consiste em construir objetos em etapas, ou seja temos um objeto objeto, normalmente chamado de `builder` que nos permite construir de pouco em pouco um objeto e após a conclusão efetivamos a construção do objeto criando uma nova instancia do objeto sendo construido.

Um bom exemplo onde este `pattern` é extensivelmente usado é com [protobufs](https://protobuf.dev/). Onde podemos definir objetos em uma lingaguem neutra e compilarmos (gerar definições) para multiplas linguagens diferentes, assim nos permitindo compartilhar nosso objetos em diferentes linguagens, pois estes objetos podem ser serializados e deserializados. 

Este `pattern` nos permite criar objetos complexos de forma mais simples e até mesmo reutilizar este `builder` em um ponto futuro para criar objetos similares porem com algumas pequena modificações. Segue um pequeno exemplo de um `builder pattern` em `Rust`.

```rust
let person = PersonBuilder::new()
            .name("Gabriel")
            .age(22)
            .college("UFABC")
            ... //set/add new properties to the object
            .build();
```

Porem não estamos aqui para falar de `Rust`, mas sim `Haskell`. Primeiramente vamos construir um pequeno `builder pattern` em `Haskell`, onde vamos encontrar multiplos pontos de potenciais falhas ou erros, em seguida vamos entender como podemos melhorar este `pattern` usando os tipos a nosso favor para remover todos os potencias erros durante a execução e transforma-los em erros de compilação, assim nos garantido a correta execução do nosso programa. Para atingir isto vamos usar `TypeStates`.

## Parte 1 - Builder Pattern em Haskell

Durante este tutorial vamos usar como exemplo o objeto `Person` onde teremos alguns campos obrigatorios, alguns campos opcionais, alguns campos com valores específicos. Vamos começar definido nosso `data type` em `Haskell`. (Este exemplo é uma modificação do [objeto definido no tutorial sobre protobufs em c++](https://protobuf.dev/getting-started/cpptutorial/#protocol-format))

```Haskell
data Person = Person {
    id :: Int,
    name :: String,
    access :: String,
    email :: Maybe String,
    phones :: [(String, String)]
} deriving (Show)
```

Vamos supor que o campo `access` deve assumir somente os valores `"sudo", "admin", "common"`, é facil de perceber que usar uma `String` para representar estes valores não é ergonomico, pois nada nos impede de usar um valor invalido, como por exemplo `"eu sou um valor invalido"`.

Para isto podemos um `Tipo Soma` ou seja.

```Haskell
data Access = Sudo | Admin | Common
    deriving (Show)
```

Agora se usarmos o tipo `Access` ao inves de uma `String`, temos a garantia que os valores utilizados serão bem definidos.

Vamos fazer o mesmo para o primeiro campo da tupla de telefones, pois este valor demonstra qual tipo de telefone.

```Haskell
data PhoneType = Mobile | Work | Home
    deriving (Show)
```

Logo nosso novo objeto `Person` pode ser definido como.

```Haskell
data Person = Person {
    idNum :: Int,
    name :: String,
    access :: Access,
    email :: Maybe String,
    phones :: [(PhoneType, String)]
} deriving (Show)
```

Com isto vamos definir nosso `builder`, para que possamos criar objetos do tipo `Person`.

```Haskell
data PersonBuilder = PersonBuilder {
    idNum' :: Maybe Int,
    name' :: Maybe String,
    access' :: Maybe Access,
    email' :: Maybe String,
    phones' :: [(PhoneType, String)]
} deriving (Show)

new :: PersonBuilder
new = PersonBuilder Nothing Nothing Nothing Nothing []
```

Todos os campos são definido como `Maybe a`, pois ao criarmos o `builder`, nenhum destes campos serão definidos, porém com o tempo iremos definindo estes valores. Vamos agora definir nossas funções para agir sobre nosso `builder`.

```Haskell
setIdNum :: Int -> PersonBuilder -> PersonBuilder
setIdNum idNum builder = builder { idNum' = Just idNum }

setName :: String -> PersonBuilder -> PersonBuilder
setName name builder = builder { name' = Just name }

setAccess :: Access -> PersonBuilder -> PersonBuilder
setAccess access builder = builder { access' = Just access }

setEmail :: String -> PersonBuilder -> PersonBuilder
setEmail email builder = builder { email' = Just email }

addPhone :: PhoneType -> String -> PersonBuilder -> PersonBuilder
addPhone t phone builder = builder { phones' = (t, phone) : phones' builder }
```

Com isso vamos definir nossa função `build`, temos 3 opções:
1. Causar um erro durante a contrução, levando a terminação do programa (não é solução mais ideal, pois causar terminação repentina nunca é legal)
2. Retornar um `Maybe Person` (também não é solução mais ideal, pois ao retornar um `Maybe` não é possivel saber qual foi o problema que causou a variante `Nothing`)
3. Retornar um `Either PersonBuilderError Person` (utilizaremos está opção, pois podemos colocar na variante `Left` o erro que causou o erro na construção do objeto).

(É a partir deste ponto onde as coisas começam a ficar feias).

Primeiro vamos definir o tipo de erro:

```Haskell
data PersonBuilderError = MissingIdNum | MissingName | MissingAccess
    deriving (Show)
```

Agora vamos definir a função build:

```Haskell
maybeToEither :: a -> Maybe b -> Either a b
maybeToEither err = maybe (Left err) Right

build :: PersonBuilder -> Either PersonBuilderError Person
build builder = do
    idNum <- maybeToEither MissingIdNum (idNum' builder)
    name <- maybeToEither MissingName (name' builder)
    access <- maybeToEither MissingAccess (access' builder)
    Right Person { idNum, name, access, email = email' builder, phones = phones' builder }
```

Primeiro temos a função `maybeToEither` que mapeia `Maybe` para um `Either`. Devido ao uso `do block` este código fica relativamente legivel, porém o mesmo ainda é proeminente a erros durante a execução, pois o usuario pode deixar de definir um dos campos necessarios. Por exemplo a saída de:

```Haskell
main :: IO ()
main = do
    let p0 = build $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p1 = build $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p2 = build $ addPhone Mobile "99999-9999" $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p3 = build $ setName "Gabriel" new
    let p4 = build $ setIdNum 0 new
    let p5 = build $ setName "Gabriel" $ setIdNum 0 new
    print p0
    print p1
    print p2
    print p3
    print p4
    print p5
```

```
Right (Person {idNum = 0, name = "Gabriel", access = Sudo, email = Nothing, phones = []})
Right (Person {idNum = 0, name = "Gabriel", access = Sudo, email = Just "email@email.com", phones = []})
Right (Person {idNum = 0, name = "Gabriel", access = Sudo, email = Just "email@email.com", phones = [(Mobile,"99999-9999")]})
Left MissingIdNum
Left MissingName
Left MissingAccess
```

Ou seja após a construção do objeto ainda temos que verificar por erros, o que não é ideal.

<details>
<summary>Código completo</summary>

```Haskell
module Main where

data Access = Sudo | Admin | Common
    deriving (Show)

data PhoneType = Mobile | Work | Home
    deriving (Show)

data Person = Person {
    idNum :: Int,
    name :: String,
    access :: Access,
    email :: Maybe String,
    phones :: [(PhoneType, String)]
} deriving (Show)

data PersonBuilder = PersonBuilder {
    idNum' :: Maybe Int,
    name' :: Maybe String,
    access' :: Maybe Access,
    email' :: Maybe String,
    phones' :: [(PhoneType, String)]
} deriving (Show)

data PersonBuilderError = MissingIdNum | MissingName | MissingAccess
    deriving (Show)

new :: PersonBuilder
new = PersonBuilder Nothing Nothing Nothing Nothing []

setIdNum :: Int -> PersonBuilder -> PersonBuilder
setIdNum idNum builder = builder { idNum' = Just idNum }

setName :: String -> PersonBuilder -> PersonBuilder
setName name builder = builder { name' = Just name }

setAccess :: Access -> PersonBuilder -> PersonBuilder
setAccess access builder = builder { access' = Just access }

setEmail :: String -> PersonBuilder -> PersonBuilder
setEmail email builder = builder { email' = Just email }

addPhone :: PhoneType -> String -> PersonBuilder -> PersonBuilder
addPhone t phone builder = builder { phones' = (t, phone) : phones' builder }

maybeToEither :: a -> Maybe b -> Either a b
maybeToEither err = maybe (Left err) Right

build :: PersonBuilder -> Either PersonBuilderError Person
build builder = do
    idNum <- maybeToEither MissingIdNum (idNum' builder)
    name <- maybeToEither MissingName (name' builder)
    access <- maybeToEither MissingAccess (access' builder)
    Right Person { idNum, name, access, email = email' builder, phones = phones' builder }

main :: IO ()
main = do
    let p0 = build $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p1 = build $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p2 = build $ addPhone Mobile "99999-9999" $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p3 = build $ setName "Gabriel" new
    let p4 = build $ setIdNum 0 new
    let p5 = build $ setName "Gabriel" $ setIdNum 0 new
    print p0
    print p1
    print p2
    print p3
    print p4
    print p5
```
</details>

## Parte 2 - TypeState

Imagine se conseguissemos usar o sistema de tipos para representar estados e transições entre estados (similar a um estado de maquina). Se conseguissemos fazer isso poderiamos apenas aplicar funções que estão bem definidas para estados específicos, por exemplo apenas chamar a função `build` caso o objeto tenha sido construido de forma correta.

Felizmente isso é possivel, para atingir nosso objetivo podemos usar tipos genericos para representar estados e pattern matching em funções (sobre os tipos) para que as mesmas possam ser aplicadas em estados (tipos) específicos.

Vamos declarar nosso tipos que representam nossos estados desejados:

```Haskell
data NoIdNum = NoIdNum deriving (Show)
data WithIdNum = WithIdNum Int deriving (Show)

data NoName = NoName deriving (Show)
data WithName = WithName String deriving (Show)

data NoAccess = NoAccess deriving (Show)
data WithAccess = WithAccess Access deriving (Show)

data NoSeal = NoSeal deriving (Show)
data WithSeal = WithSeal deriving (Show)
```

Como podemos ver nossos estados são declarados em pares, um estado que representa a falta do campo e um estado que representa a definição do campo. O estado `Seal` será usado no futuro para que possamos `"selar"` nosso builder, assim evitando modificações do mesmo, pois ao selarmos nosso `builder`, nenhuma função que modifica o `builder` estará definida para o estado `WithSeal`. Este estado será usado como um `tipo fantasma`, pois o mesmo so representa uma espécie de `"metadata"` do objeto.

Com isso vamos declara nosso novo objeto `Person`:

```Haskell
data PersonBuilder idNum name access seal = PersonBuilder {
    idNum' :: idNum,
    name' :: name,
    access' :: access,
    email' :: Maybe String,
    phones' :: [(PhoneType, String)]
} deriving (Show)

new :: PersonBuilder NoIdNum NoName NoAccess NoSeal
new = PersonBuilder NoIdNum NoName NoAccess Nothing []
```

Logo nosso novo tipo `Person` é genérico sobre os estados e a criação de um `builder` com a função `new` retornar um `PersonBuilder` onde os tipos são do tipo `No*`, pois nenhum campo foi `setado` ainda, também usamos o tipo genérico `NoSeal`, pois o objeto ainda não foi `"selado"` e queremos permitir a modificação deste `builder`.

Agora as funções que agem sobre o objeto `builder`:

```Haskell
setIdNum :: Int -> PersonBuilder idNum name access NoSeal -> PersonBuilder WithIdNum name access NoSeal
setIdNum idNum builder = builder { idNum' = WithIdNum idNum }
```

A função `setIdNum` é genérica sobre qualquer um dos estados exceto o `Seal`, pois o objeto não deve ter sido `"selado"` ainda para que possamos modifica-lo. Ao `setarmos` o campo `idNum` o novo objeto retornado passa a ser do tipo `PersonBuilder WithIdNum ... NoSeal`, ou seja transicionamos do estado `NoIdNum -> WithIdNum`. O `builder` de entrada pode ser genérico sobre o estado `IdNum`, pois isso permite usar a função `setIdNum` multiplas vezes para redefinir o campo `idNum`. Caso o builder de entrada não fosse genérico (`PersonBuilder NoIdNum ...`) não seria possivel chamar a função novamente, pois o novo `builder` é do tipo `PersonBuilder WithIdNum ...`.

Similarmente o mesmo pode ser feito para as outras funções.

```Haskell
setName :: String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum WithName access NoSeal
setName name builder = builder { name' = WithName name }

setAccess :: Access -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name WithAccess NoSeal
setAccess access builder = builder { access' = WithAccess access }

setEmail :: String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access NoSeal
setEmail email builder = builder { email' = Just email }

addPhone :: PhoneType -> String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access NoSeal
addPhone t phone builder = builder { phones' = (t, phone) : phones' builder }
```

Como os campos `email` e `phones` não precisam de estados para representa-los, os mesmo aceitam objeto genérico sobre todos os estados e retorna um objeto sobre os mesmo estados de entrada, pois não transicionamos de nenhum estado para nenhum estado.

Vamos definir a função `seal`, que `"selará"` nosso objeto para evitar modificações. Onde vamos sair do estado `NoSeal -> WithSeal`.

```Haskell
seal :: PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access WithSeal
seal builder = builder { idNum' = idNum' builder, name' = name' builder, access' = access' builder, email' = email' builder, phones' = phones' builder }
```

Como o estado que representa o `Seal` é fantasma, nao podemos definir a função como `seal builder = builder`, pois os tipos são diferentes, logo criamos um novo builder onde os campos são exatamente os mesmos e deixamos o compilador lidar com o resto. (Nota: Não sei se existe uma forma mais compacta de representar isso).

Já nosso método `build` pode ser definido como:

```Haskell
build :: PersonBuilder WithIdNum WithName WithAccess seal -> Person
build builder = Person { idNum = getIdNum (idNum' builder), name = getName (name' builder), access = getAccess (access' builder), email = email' builder, phones = phones' builder }
    where
        getIdNum :: WithIdNum -> Int
        getIdNum (WithIdNum idNum) = idNum

        getName :: WithName -> String
        getName (WithName name) = name

        getAccess :: WithAccess -> Access
        getAccess (WithAccess access) = access
```

A função `build` está definidia apenas para os estados `WithIdNum WithName WithAccess`, pois isso nos garante que estes campos foram `setados`, logo não precisamos checar durante a execução. Esta função pode ser genérica sobre o estado `Seal`, pois independe se o `builder` esta `"selado"` ou não para construimos o objeto final (lembrando que o `"selo"` so evita a modificação do builder). So precisamos retirar os valores dos estados `With*`, por isso definimos as funções `get*`.

Agora vamos testar:

```Haskell
main :: IO ()
main = do
    let p0 = build $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p1 = build $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p2 = build $ addPhone Mobile "99999-9999" $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    print p0
    print p1
    print p2
```

Ainda funciona como o esperado.

```
Person {idNum = 0, name = "Gabriel", access = Sudo, email = Nothing, phones = []}
Person {idNum = 0, name = "Gabriel", access = Sudo, email = Just "email@email.com", phones = []}
Person {idNum = 0, name = "Gabriel", access = Sudo, email = Just "email@email.com", phones = [(Mobile,"99999-9999")]}
```

Porém agora vamos cometer alguns erros de forma proposital e ver o que o compilador nos diz (lembrando que ao cometermos erros o programa nem mesmo compilará):

```Haskell
main :: IO ()
main = do
    let p0 = build $ setName "Gabriel" new
    let p1 = build $ setIdNum 0 new
    let p2 = build $ setName "Gabriel" $ setIdNum 0 new
    print p0
    print p1
    print p2
```

Erros do compialdor:

```
<source>:72:40: error: [GHC-83865]
    * Couldn't match type `NoIdNum' with `WithIdNum'
      Expected: PersonBuilder WithIdNum NoName WithAccess NoSeal
        Actual: PersonBuilder NoIdNum NoName NoAccess NoSeal
    * In the second argument of `setName', namely `new'
      In the second argument of `($)', namely `setName "Gabriel" new'
      In the expression: build $ setName "Gabriel" new
   |
72 |     let p0 = build $ setName "Gabriel" new
   |                                        ^^^

<source>:73:33: error: [GHC-83865]
    * Couldn't match type `NoName' with `WithName'
      Expected: PersonBuilder NoIdNum WithName WithAccess NoSeal
        Actual: PersonBuilder NoIdNum NoName NoAccess NoSeal
    * In the second argument of `setIdNum', namely `new'
      In the second argument of `($)', namely `setIdNum 0 new'
      In the expression: build $ setIdNum 0 new
   |
73 |     let p1 = build $ setIdNum 0 new
   |                                 ^^^

<source>:74:53: error: [GHC-83865]
    * Couldn't match type `NoAccess' with `WithAccess'
      Expected: PersonBuilder NoIdNum NoName WithAccess NoSeal
        Actual: PersonBuilder NoIdNum NoName NoAccess NoSeal
    * In the second argument of `setIdNum', namely `new'
      In the second argument of `($)', namely `setIdNum 0 new'
      In the second argument of `($)', namely
        `setName "Gabriel" $ setIdNum 0 new'
   |
74 |     let p2 = build $ setName "Gabriel" $ setIdNum 0 new
   |                                                     ^^^
Compiler returned: 1
```

Agora vamos testar redefinir um campo (lembrando que isto é permitido):

```Haskell
main :: IO ()
main = do
    let b0 = setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let b1 = setIdNum 1 b0
    let p0 = build b1
    print p0
```

```
Person {idNum = 1, name = "Gabriel", access = Sudo, email = Nothing, phones = []}
```

Vamos `"selar"` um `builder` e tentar modificar algum campo (lembrando que isto não é permitido):

```Haskell
main :: IO ()
main = do
    let b0 = seal $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let b1 = setIdNum 1 b0
    let p0 = build b1
    print p0
```

```
<source>:73:25: error: [GHC-83865]
    * Couldn't match type `WithSeal' with `NoSeal'
      Expected: PersonBuilder WithIdNum WithName WithAccess NoSeal
        Actual: PersonBuilder WithIdNum WithName WithAccess WithSeal
    * In the second argument of `setIdNum', namely `b0'
      In the expression: setIdNum 1 b0
      In an equation for `b1': b1 = setIdNum 1 b0
   |
73 |     let b1 = setIdNum 1 b0
   |                         ^^
Compiler returned: 1
```

E por último vamos chamar o método `build` em um `builder` `"selado"` (lembrando que isto é permitido):

```Haskell
main :: IO ()
main = do
    let p0 = build $ seal $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    print p0
```

```
Person {idNum = 0, name = "Gabriel", access = Sudo, email = Nothing, phones = []}
```

<details>
<summary>Código completo (descomentar as versões da função main)</summary>

```Haskell
module Main where

data Access = Sudo | Admin | Common
    deriving (Show)

data PhoneType = Mobile | Work | Home
    deriving (Show)

data Person = Person {
    idNum :: Int,
    name :: String,
    access :: Access,
    email :: Maybe String,
    phones :: [(PhoneType, String)]
} deriving (Show)

data NoIdNum = NoIdNum deriving (Show)
data WithIdNum = WithIdNum Int deriving (Show)

data NoName = NoName deriving (Show)
data WithName = WithName String deriving (Show)

data NoAccess = NoAccess deriving (Show)
data WithAccess = WithAccess Access deriving (Show)

data NoSeal = NoSeal deriving (Show)
data WithSeal = WithSeal deriving (Show)

data PersonBuilder idNum name access seal = PersonBuilder {
    idNum' :: idNum,
    name' :: name,
    access' :: access,
    email' :: Maybe String,
    phones' :: [(PhoneType, String)]
} deriving (Show)

new :: PersonBuilder NoIdNum NoName NoAccess NoSeal
new = PersonBuilder NoIdNum NoName NoAccess Nothing []

setIdNum :: Int -> PersonBuilder idNum name access NoSeal -> PersonBuilder WithIdNum name access NoSeal
setIdNum idNum builder = builder { idNum' = WithIdNum idNum }

setName :: String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum WithName access NoSeal
setName name builder = builder { name' = WithName name }

setAccess :: Access -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name WithAccess NoSeal
setAccess access builder = builder { access' = WithAccess access }

setEmail :: String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access NoSeal
setEmail email builder = builder { email' = Just email }

addPhone :: PhoneType -> String -> PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access NoSeal
addPhone t phone builder = builder { phones' = (t, phone) : phones' builder }

seal :: PersonBuilder idNum name access NoSeal -> PersonBuilder idNum name access WithSeal
seal builder = builder { idNum' = idNum' builder, name' = name' builder, access' = access' builder, email' = email' builder, phones' = phones' builder }

build :: PersonBuilder WithIdNum WithName WithAccess seal -> Person
build builder = Person { idNum = getIdNum (idNum' builder), name = getName (name' builder), access = getAccess (access' builder), email = email' builder, phones = phones' builder }
    where
        getIdNum :: WithIdNum -> Int
        getIdNum (WithIdNum idNum) = idNum

        getName :: WithName -> String
        getName (WithName name) = name

        getAccess :: WithAccess -> Access
        getAccess (WithAccess access) = access

-- Sem Erros
main :: IO ()
main = do
    let p0 = build $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p1 = build $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    let p2 = build $ addPhone Mobile "99999-9999" $ setEmail "email@email.com" $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
    print p0
    print p1
    print p2

-- Com Erros (Campos Faltando)
-- main :: IO ()
-- main = do
--     let p0 = build $ setName "Gabriel" new
--     let p1 = build $ setIdNum 0 new
--     let p2 = build $ setName "Gabriel" $ setIdNum 0 new
--     print p0
--     print p1
--     print p2

-- Sem Erros (Redefinindo Campo)
-- main :: IO ()
-- main = do
--     let b0 = setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
--     let b1 = setIdNum 1 b0
--     let p0 = build b1
--     print p0

-- Com Erros (Redefinindo Campo Selado)
-- main :: IO ()
-- main = do
--     let b0 = seal $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
--     let b1 = setIdNum 1 b0
--     let p0 = build b1
--     print p0

-- Sem Erros (Buildando Objeto Selado)
-- main :: IO ()
-- main = do
--     let p0 = build $ seal $ setAccess Sudo $ setName "Gabriel" $ setIdNum 0 new
--     print p0
```
</details>