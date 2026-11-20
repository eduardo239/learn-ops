# Java: do zero ao profissional — Guia completo

Nov 20, 2026 · @Oi

## 1. Introdução ao Java

Java é uma linguagem orientada a objetos, fortemente tipada e compilada para **bytecode**, que roda em qualquer sistema com uma **JVM** (Java Virtual Machine). Este guia usa o **Java 25**, a versão LTS (suporte de longo prazo) mais recente, lançada em setembro de 2025 ([Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)).

### 1.1 O que é e por que aprender

Criado pela Sun Microsystems em 1995 e mantido pela Oracle desde 2010, o Java nasceu com o lema *“escreva uma vez, rode em qualquer lugar”*. O código não é compilado para um processador específico, e sim para bytecode, que a JVM traduz para a máquina em que está rodando.

Hoje o Java está em sistemas bancários, e-commerce, back-ends de grandes empresas, Android (junto com Kotlin), big data (Kafka, Spark, Hadoop, Elasticsearch) e ferramentas como IntelliJ e Minecraft. Os motivos:

- **Estabilidade e compatibilidade:** código escrito há 20 anos ainda compila e roda.
- **Desempenho:** o compilador JIT otimiza o código em tempo de execução, chegando perto de linguagens nativas.
- **Ecossistema enorme:** Spring, Hibernate, Maven, JUnit e milhares de bibliotecas maduras.
- **Mercado de trabalho:** uma das linguagens mais pedidas em vagas de back-end, no Brasil e no exterior.
- **Evolução constante:** uma versão nova a cada seis meses, com records, pattern matching e virtual threads modernizando a linguagem.

### 1.2 JDK, JRE e JVM

&#91;embedded content: Do código-fonte à execução · JDK, JRE e JVM\]

O **JDK** é o que você instala para programar: inclui o compilador `javac`, o comando `java` e as ferramentas. A **JVM** é a máquina virtual que executa o bytecode, gerencia a memória com o *garbage collector* e compila trechos quentes para código nativo com o **JIT** (*Just-In-Time*). Desde o Java 11 não existe mais um download separado de JRE: você instala o JDK e, para distribuir, pode gerar um runtime enxuto com `jlink`.

### 1.3 Versões e ciclo de lançamento

Uma nova versão sai a cada seis meses (março e setembro). A cada dois anos, uma delas é **LTS**, com anos de atualizações de segurança. Em empresas, use sempre uma LTS.

| Versão | Lançamento | Tipo | Destaques |
| --- | --- | --- | --- |
| 29 | setembro de 2027 (planejada) | LTS | Próxima LTS |
| 27 | setembro de 2026 | Não LTS | Versão mais recente |
| **25** | setembro de 2025 | **LTS** | Arquivos-fonte compactos, `main` simplificado, *scoped values* |
| 21 | setembro de 2023 | LTS | Virtual threads, pattern matching para `switch`, *record patterns* |
| 17 | setembro de 2021 | LTS | *Sealed classes*, records consolidados |
| 11 | setembro de 2018 | LTS | `var` em lambdas, cliente HTTP nativo |
| 8 | março de 2014 | LTS | Lambdas, Streams, `java.time` |

### 1.4 Distribuições do JDK

O código do Java é aberto (projeto **OpenJDK**), e várias empresas distribuem builds compatíveis e gratuitos. Para estudar e para a maioria dos projetos, qualquer uma serve:

| Distribuição | Mantenedor | Observação |
| --- | --- | --- |
| Eclipse Temurin | Adoptium (Eclipse Foundation) | Opção neutra e muito usada; recomendada neste guia |
| Oracle JDK | Oracle | Gratuito sob os termos da Oracle a partir do 21; verifique a licença para uso comercial |
| Amazon Corretto | AWS | Otimizado para a AWS, gratuito |
| Microsoft Build of OpenJDK | Microsoft | Usado no Azure |
| Azul Zulu | Azul | Builds para muitas plataformas |

### 1.5 Como usar este guia

Os capítulos são progressivos: do capítulo 2 ao 9 você aprende a linguagem; do 10 ao 14, as ferramentas e o funcionamento interno; no 15 e no 16, o ecossistema profissional e um projeto final. Cada capítulo termina com **exercícios**. Digite os exemplos, execute-os e altere-os: é assim que a sintaxe fixa.

1. Introdução ao Java
2. Preparando o ambiente
3. Fundamentos da linguagem
4. Métodos, arrays e Strings
5. Orientação a objetos
6. Java moderno: records, enums, sealed e pattern matching
7. Exceções
8. Generics e Collections
9. Programação funcional: lambdas, Streams e Optional
10. Arquivos e datas
11. Concorrência
12. Build e organização de projetos: Maven e Gradle
13. Testes automatizados
14. Por dentro da JVM
15. Ecossistema: JDBC, JPA e Spring Boot
16. Projeto final, boas práticas e trilha de estudos
17. Apêndices: guia rápido e glossário

## 2. Preparando o ambiente

Você precisa de duas coisas: um **JDK 25** e um editor ou IDE. Com isso, consegue compilar e rodar o primeiro programa em menos de dez minutos.

### 2.1 Instalando o JDK

A forma mais prática no Linux e no macOS é o **SDKMAN!**, que instala e alterna entre versões do Java (e também de Maven e Gradle):

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk list java              # lista as versões disponíveis
sdk install java 25.0.x-tem   # troque x pela versão mais recente da lista (tem = Temurin)
sdk default java 25.0.x-tem
```

Alternativas por sistema:

| Sistema | Comando |
| --- | --- |
| Ubuntu e Debian recentes | `sudo apt install openjdk-25-jdk` |
| Fedora | `sudo dnf install java-25-openjdk-devel` |
| macOS (Homebrew) | `brew install --cask temurin@25` |
| Windows (winget) | `winget install EclipseAdoptium.Temurin.25.JDK` |

No Windows, o instalador da Temurin oferece a opção de configurar `JAVA_HOME` e o `PATH`. Marque-a. Depois, confira em um terminal novo:

```bash
java -version
javac -version
```

As duas devem mostrar a versão 25. Se aparecer uma versão antiga, há outro JDK antes no `PATH`: ajuste a variável `JAVA_HOME` para a pasta do JDK 25 e coloque `$JAVA_HOME/bin` no início do `PATH`.

### 2.2 Escolhendo uma IDE

| Ferramenta | Perfil |
| --- | --- |
| IntelliJ IDEA (Community ou Ultimate) | A mais usada no mercado para Java; refatoração e análise de código excelentes |
| VS Code + Extension Pack for Java | Leve, ótima para quem já usa VS Code em outras linguagens |
| Eclipse IDE | Tradicional, gratuita, ainda comum em empresas |

Para aprender, recomendamos o **IntelliJ IDEA Community**. Nos primeiros capítulos, porém, use também o terminal: entender `javac` e `java` evita muita confusão depois.

### 2.3 O primeiro programa

Crie o arquivo `OlaMundo.java`. Em Java, uma classe pública fica em um arquivo com exatamente o mesmo nome:

```java
public class OlaMundo {
    public static void main(String[] args) {
        System.out.println("Olá, mundo!");
    }
}
```

- `public class OlaMundo` declara uma classe, a unidade básica de código em Java.
- `public static void main(String[] args)` é o **ponto de entrada**: a JVM começa a execução por esse método.
- `System.out.println` imprime uma linha na saída padrão.

Compile e execute:

```bash
javac OlaMundo.java    # gera OlaMundo.class (bytecode)
java OlaMundo          # executa a classe na JVM
```

Para programas de um arquivo só, o Java também compila e executa direto, sem gerar o `.class`: `java OlaMundo.java`.

### 2.4 O jeito moderno: arquivos-fonte compactos (Java 25)

A partir do Java 25, um programa simples não precisa declarar classe nem `static`. O arquivo `Ola.java` abaixo é um programa completo:

```java
void main() {
    String nome = IO.readln("Qual é o seu nome? ");
    IO.println("Olá, " + nome + "!");
}
```

Rode com `java Ola.java`. A classe `IO` traz `println`, `print` e `readln` para entrada e saída simples no console. Esse formato é ótimo para estudar e para scripts; em projetos, você continuará usando classes com pacotes, como no restante do guia.

### 2.5 jshell: experimente sem criar arquivos

O `jshell` é um terminal interativo (REPL) que executa trechos de Java na hora. Use-o para testar uma expressão ou um método da biblioteca:

```text
$ jshell
jshell> int x = 21 * 2
x ==> 42
jshell> "java".toUpperCase()
$2 ==> "JAVA"
jshell> /exit
```

### Exercícios

- [ ] Instale o JDK 25 e confirme as versões de `java` e `javac`.
- [ ] Escreva, compile e execute o `OlaMundo` pelo terminal; depois, apague o `.class` e rode com `java OlaMundo.java`.
- [ ] Crie um programa compacto que leia dois números com `IO.readln` e imprima a soma (dica: `Integer.parseInt`).
- [ ] No jshell, descubra o que retorna `Math.max(7, 3)` e `"banana".length()`.

## 3. Fundamentos da linguagem

Java é **estaticamente tipado**: todo valor tem um tipo conhecido em tempo de compilação, e o compilador recusa operações incompatíveis antes de o programa rodar. Este capítulo cobre tipos, variáveis, operadores e controle de fluxo.

### 3.1 Tipos primitivos

Existem oito tipos primitivos. Eles guardam o valor diretamente e não são objetos.

| Tipo | Tamanho | Faixa ou uso | Exemplo |
| --- | --- | --- | --- |
| `byte` | 8 bits | −128 a 127 | `byte b = 10;` |
| `short` | 16 bits | −32.768 a 32.767 | `short s = 1000;` |
| `int` | 32 bits | cerca de ±2,1 bilhões; o padrão para inteiros | `int idade = 30;` |
| `long` | 64 bits | inteiros muito grandes; sufixo `L` | `long populacao = 8_000_000_000L;` |
| `float` | 32 bits | decimal de precisão simples; sufixo `f` | `float f = 1.5f;` |
| `double` | 64 bits | decimal de precisão dupla; o padrão para decimais | `double pi = 3.14159;` |
| `char` | 16 bits | um caractere Unicode, entre aspas simples | `char letra = 'A';` |
| `boolean` | — | `true` ou `false` | `boolean ativo = true;` |

O sublinhado em números (`1_000_000`) só melhora a leitura. Todo o resto — `String`, arrays, listas, suas classes — são **tipos de referência**: a variável guarda uma referência para um objeto, que pode ser `null`.

Para cada primitivo existe uma classe **wrapper** (`Integer`, `Long`, `Double`, `Boolean`, `Character` etc.), usada onde só cabem objetos, como em coleções. A conversão entre os dois é automática (*autoboxing* e *unboxing*).

> **Atenção:** nunca use `double` para dinheiro. `0.1 + 0.2` resulta em `0.30000000000000004`. Para valores monetários, use `BigDecimal` (capítulo 4) ou centavos em `long`.

### 3.2 Variáveis, constantes e `var`

```java
int quantidade = 5;               // declaração com tipo explícito
final double TAXA = 0.15;         // final: não pode ser reatribuída
var nome = "Ana";                 // o compilador infere String
var precos = new ArrayList<Double>();  // infere ArrayList<Double>
```

`var` só funciona em variáveis locais inicializadas na declaração. O tipo continua estático: depois de `var nome = "Ana"`, `nome` é `String` para sempre. Use `var` quando o tipo estiver óbvio pelo lado direito.

Convenções de nomes: variáveis e métodos em `camelCase`, classes em `PascalCase`, constantes em `MAIUSCULAS_COM_SUBLINHADO` e pacotes em minúsculas (`br.com.empresa.loja`).

### 3.3 Operadores

| Categoria | Operadores | Observação |
| --- | --- | --- |
| Aritméticos | `+ - * / %` | `7 / 2` é `3` (divisão inteira); `7 % 2` é `1` (resto) |
| Atribuição | `= += -= *= /= %=` | `x += 5` equivale a `x = x + 5` |
| Incremento | `++ --` | `i++` usa e depois incrementa; `++i` incrementa e depois usa |
| Comparação | `== != > < >= <=` | Para objetos, `==` compara referências, não conteúdo |
| Lógicos | `&& \|\| !` | Curto-circuito: `&&` não avalia o lado direito se o esquerdo for falso |
| Ternário | `cond ? a : b` | `String s = idade >= 18 ? "adulto" : "menor";` |

### 3.4 Conversões de tipo

```java
int i = 100;
long l = i;                    // alargamento: automático e seguro
double d = 9.78;
int truncado = (int) d;        // estreitamento: exige cast; resulta em 9

int n = Integer.parseInt("42");      // String para int
double x = Double.parseDouble("3.5");
String texto = String.valueOf(n);    // int para String
```

### 3.5 Controle de fluxo

**if / else:**

```java
if (nota >= 7) {
    IO.println("Aprovado");
} else if (nota >= 5) {
    IO.println("Recuperação");
} else {
    IO.println("Reprovado");
}
```

**switch moderno** (expressão com setas, sem `break` e sem “queda” entre casos):

```java
String tipo = switch (diaDaSemana) {
    case 1, 7 -> "Fim de semana";
    case 2, 3, 4, 5, 6 -> "Dia útil";
    default -> throw new IllegalArgumentException("Dia inválido: " + diaDaSemana);
};
```

Quando um caso precisa de várias linhas, use um bloco e `yield` para devolver o valor: `case 1 -> { log(); yield "Domingo"; }`.

**Laços:**

```java
for (int i = 0; i < 5; i++) {         // for clássico
    IO.println("i = " + i);
}

for (String fruta : List.of("maçã", "uva")) {   // for-each
    IO.println(fruta);
}

int tentativas = 0;
while (tentativas < 3) {              // while: testa antes
    tentativas++;
}

do {                                  // do-while: executa ao menos uma vez
    tentativas--;
} while (tentativas > 0);
```

`break` sai do laço; `continue` pula para a próxima iteração.

### Exercícios

- [ ] Escreva um programa que converta uma temperatura de Celsius para Fahrenheit (F = C × 9/5 + 32). Teste com `int` e com `double` e explique a diferença.
- [ ] Imprima a tabuada de 1 a 10 de um número lido do teclado.
- [ ] Use um `switch` com setas para mostrar o nome do mês a partir do número.
- [ ] Implemente o FizzBuzz de 1 a 100: “Fizz” para múltiplos de 3, “Buzz” para múltiplos de 5 e “FizzBuzz” para múltiplos de ambos.

## 4. Métodos, arrays e Strings

Métodos organizam o código em blocos reutilizáveis; arrays guardam sequências de tamanho fixo; e `String` é o tipo que você mais vai usar no dia a dia.

### 4.1 Declarando métodos

```java
//  modificadores  retorno  nome      parâmetros
    public static  double   media(double a, double b) {
        return (a + b) / 2;
    }
```

- O **tipo de retorno** é obrigatório; use `void` quando o método não devolve nada.
- `static` indica que o método pertence à classe, e não a um objeto (capítulo 5).
- **Sobrecarga**: vários métodos com o mesmo nome e parâmetros diferentes.
- **Varargs**: `int soma(int... numeros)` aceita qualquer quantidade de argumentos, recebidos como array.

```java
static int soma(int a, int b) { return a + b; }
static double soma(double a, double b) { return a + b; }   // sobrecarga
static int soma(int... numeros) {                         // varargs
    int total = 0;
    for (int n : numeros) total += n;
    return total;
}
```

### 4.2 Java sempre passa por valor

Quando você chama um método, o Java **copia** o valor do argumento. Para primitivos, copia o número. Para objetos, copia a **referência**: o método pode alterar o objeto apontado, mas não consegue fazer a variável de quem chamou apontar para outro objeto.

```java
static void dobra(int x) { x = x * 2; }
static void adiciona(List<String> lista) { lista.add("novo"); }

int n = 5;
dobra(n);            // n continua 5
var itens = new ArrayList<String>();
adiciona(itens);     // itens agora tem "novo": o objeto foi alterado
```

### 4.3 Arrays

```java
int[] notas = new int[5];                 // 5 posições, todas 0
String[] dias = {"seg", "ter", "qua"};    // inicialização direta
notas[0] = 8;
int tamanho = dias.length;                // 3 (atributo, sem parênteses)

int[][] matriz = { {1, 2}, {3, 4} };       // array bidimensional
int valor = matriz[1][0];                 // 3

Arrays.sort(notas);                       // ordena
IO.println(Arrays.toString(notas));       // [0, 0, 0, 0, 8]
```

Arrays têm **tamanho fixo** e índices de `0` a `length - 1`; acessar fora disso lança `ArrayIndexOutOfBoundsException`. Quando o tamanho varia, prefira `List` (capítulo 8).

### 4.4 Strings

`String` é **imutável**: todo método que “altera” uma String devolve uma nova.

| Método | Exemplo | Resultado |
| --- | --- | --- |
| `length()` | `"Java".length()` | `4` |
| `charAt(i)` | `"Java".charAt(0)` | `'J'` |
| `substring(i, f)` | `"Programa".substring(0, 4)` | `"Prog"` |
| `toUpperCase()` / `toLowerCase()` | `"java".toUpperCase()` | `"JAVA"` |
| `strip()` | `"  oi  ".strip()` | `"oi"` |
| `contains`, `startsWith`, `endsWith` | `"arquivo.txt".endsWith(".txt")` | `true` |
| `replace(a, b)` | `"a-b-c".replace("-", "/")` | `"a/b/c"` |
| `split(regex)` | `"a,b,c".split(",")` | `["a", "b", "c"]` |
| `isBlank()` | `"   ".isBlank()` | `true` |
| `repeat(n)` | `"ab".repeat(3)` | `"ababab"` |
| `String.join` | `String.join(", ", lista)` | `"x, y, z"` |
| `formatted` | `"%s tem %d anos".formatted("Ana", 30)` | `"Ana tem 30 anos"` |

> **Atenção:** compare Strings com `equals`, nunca com `==`. O operador `==` compara se são o **mesmo objeto** na memória. Use `"sim".equals(resposta)` (constante à esquerda evita `NullPointerException`) ou `equalsIgnoreCase`.

### 4.5 StringBuilder e text blocks

Concatenar Strings com `+` dentro de um laço cria um objeto novo a cada volta. Para montar texto em partes, use `StringBuilder`:

```java
var sb = new StringBuilder();
for (int i = 1; i <= 3; i++) {
    sb.append("Linha ").append(i).append('\n');
}
String resultado = sb.toString();
```

**Text blocks** (três aspas) escrevem textos de várias linhas, como JSON ou SQL, sem escapes:

```java
String sql = """
    SELECT id, nome
    FROM clientes
    WHERE cidade = ?
    ORDER BY nome
    """;
```

### 4.6 Números: Math e BigDecimal

```java
Math.max(3, 7);        // 7
Math.pow(2, 10);       // 1024.0
Math.sqrt(16);         // 4.0
Math.round(2.5);       // 3

var preco = new BigDecimal("19.90");          // sempre a partir de String
var total = preco.multiply(new BigDecimal("3"))
                 .setScale(2, RoundingMode.HALF_EVEN);   // 59.70
```

### Exercícios

- [ ] Escreva `boolean ehPalindromo(String s)` que ignore maiúsculas e espaços (“Ama la ama” é palíndromo).
- [ ] Crie um método que receba um `int[]` e devolva a média, o maior e o menor valor.
- [ ] Conte quantas vogais há em uma frase lida do teclado.
- [ ] Calcule o total de um carrinho com `BigDecimal` e compare o resultado com a mesma conta em `double`.

## 5. Orientação a objetos

Em Java, quase tudo é objeto. Uma **classe** é o molde (atributos + comportamentos) e um **objeto** é uma instância criada a partir dele com `new`. A orientação a objetos se apoia em quatro pilares: encapsulamento, herança, polimorfismo e abstração.

### 5.1 Classes, objetos e construtores

```java
public class ContaBancaria {
    private final String titular;     // atributo (estado)
    private BigDecimal saldo;

    public ContaBancaria(String titular, BigDecimal saldoInicial) {   // construtor
        if (saldoInicial.signum() < 0) {
            throw new IllegalArgumentException("Saldo inicial negativo");
        }
        this.titular = titular;      // this = o próprio objeto
        this.saldo = saldoInicial;
    }

    public ContaBancaria(String titular) {   // construtor sobrecarregado
        this(titular, BigDecimal.ZERO);      // reaproveita o outro
    }

    public void depositar(BigDecimal valor) {       // método (comportamento)
        if (valor.signum() <= 0) throw new IllegalArgumentException("Valor inválido");
        saldo = saldo.add(valor);
    }

    public BigDecimal getSaldo() { return saldo; }
    public String getTitular() { return titular; }
}
```

```java
var conta = new ContaBancaria("Ana");
conta.depositar(new BigDecimal("100.00"));
IO.println(conta.getSaldo());   // 100.00
```

Se você não escrever nenhum construtor, o compilador cria um vazio. Ao declarar qualquer construtor, esse padrão deixa de existir.

### 5.2 Encapsulamento e modificadores de acesso

**Encapsular** é esconder o estado interno e expor apenas operações válidas. No exemplo acima, ninguém consegue fazer `conta.saldo = -500`: o saldo só muda por `depositar`, que valida o valor.

| Modificador | Mesma classe | Mesmo pacote | Subclasse | Qualquer lugar |
| --- | --- | --- | --- | --- |
| `private` | sim | não | não | não |
| (nenhum) — *package-private* | sim | sim | não | não |
| `protected` | sim | sim | sim | não |
| `public` | sim | sim | sim | sim |

Regra prática: atributos `private`; métodos `public` apenas se fizerem parte do contrato da classe. Evite gerar *setters* para tudo: um setter público é uma porta aberta para estado inválido.

### 5.3 static e final

- `static`: pertence à **classe**, compartilhado por todas as instâncias. Ex.: `Math.max`, contadores, constantes.
- `final` em variável: não pode ser reatribuída. Em método: não pode ser sobrescrito. Em classe: não pode ser estendida (como `String`).

```java
public class Pedido {
    public static final int LIMITE_ITENS = 50;   // constante de classe
    private static int totalCriados = 0;          // compartilhado

    public Pedido() { totalCriados++; }
    public static int getTotalCriados() { return totalCriados; }
}
```

### 5.4 Herança

Uma classe pode **estender** outra com `extends` e herdar seus membros. Java tem herança simples: cada classe estende exatamente uma (por padrão, `Object`).

```java
public class Funcionario {
    protected final String nome;
    protected final BigDecimal salarioBase;

    public Funcionario(String nome, BigDecimal salarioBase) {
        this.nome = nome;
        this.salarioBase = salarioBase;
    }

    public BigDecimal salario() { return salarioBase; }
}

public class Gerente extends Funcionario {
    private final BigDecimal bonus;

    public Gerente(String nome, BigDecimal salarioBase, BigDecimal bonus) {
        super(nome, salarioBase);          // chama o construtor da superclasse
        this.bonus = bonus;
    }

    @Override                               // sobrescreve o comportamento
    public BigDecimal salario() {
        return super.salario().add(bonus);
    }
}
```

Use `@Override` sempre: o compilador avisa se a assinatura não corresponder a um método da superclasse.

### 5.5 Polimorfismo

**Polimorfismo** é tratar objetos de tipos diferentes pela mesma interface, e cada um responder do seu jeito. O método executado é decidido em tempo de execução, pelo tipo real do objeto:

```java
List<Funcionario> equipe = List.of(
    new Funcionario("Bruno", new BigDecimal("5000")),
    new Gerente("Carla", new BigDecimal("8000"), new BigDecimal("2000"))
);

BigDecimal folha = BigDecimal.ZERO;
for (Funcionario f : equipe) {
    folha = folha.add(f.salario());   // chama a versão certa de cada um
}
// folha = 15000
```

### 5.6 Classes abstratas e interfaces

Uma **classe abstrata** não pode ser instanciada e pode ter métodos sem corpo, que as subclasses são obrigadas a implementar. Uma **interface** define um contrato que várias classes, sem relação entre si, podem cumprir. Uma classe estende uma única classe, mas pode implementar várias interfaces.

```java
public interface MeioDePagamento {
    void pagar(BigDecimal valor);

    default String descricao() {             // método com implementação padrão
        return getClass().getSimpleName();
    }
}

public class Pix implements MeioDePagamento {
    private final String chave;
    public Pix(String chave) { this.chave = chave; }

    @Override
    public void pagar(BigDecimal valor) {
        IO.println("Pix de " + valor + " para " + chave);
    }
}

public abstract class Forma {
    public abstract double area();           // sem corpo
    public String resumo() { return "Área: " + area(); }
}
```

|  | Classe abstrata | Interface |
| --- | --- | --- |
| Estado (atributos de instância) | Pode ter | Não pode (só constantes) |
| Construtor | Pode ter | Não tem |
| Quantas uma classe pode usar | Uma (`extends`) | Várias (`implements`) |
| Quando usar | Base comum com estado e código compartilhado | Contrato ou capacidade (“pagável”, “comparável”) |

> **Dica:** prefira **composição** a herança. Em vez de `Carro extends Motor`, faça `Carro` ter um atributo `Motor`. Herança cria acoplamento forte; use-a só quando a relação “é um” for verdadeira e estável.

### 5.7 equals, hashCode e toString

Toda classe herda de `Object` os métodos `equals`, `hashCode` e `toString`. Por padrão, `equals` compara referências. Se dois objetos com os mesmos dados devem ser considerados iguais (por exemplo, para uso em `HashSet` ou como chave de `HashMap`), sobrescreva **os dois**, `equals` e `hashCode`, de forma consistente:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Cpf outro)) return false;
    return numero.equals(outro.numero);
}

@Override
public int hashCode() { return Objects.hash(numero); }

@Override
public String toString() { return "Cpf[" + numero + "]"; }
```

No próximo capítulo você verá que os **records** geram esses três métodos automaticamente.

### Exercícios

- [ ] Acrescente `sacar` à `ContaBancaria`, impedindo saldo negativo. Crie `ContaEspecial`, com limite de cheque especial, sobrescrevendo `sacar`.
- [ ] Crie a classe abstrata `Forma` com `Circulo`, `Retangulo` e `Triangulo`; some as áreas de uma lista de formas.
- [ ] Defina a interface `MeioDePagamento` com `Pix`, `Boleto` e `CartaoCredito` e um método que processe uma lista de pagamentos.
- [ ] Coloque duas instâncias “iguais” de uma classe sua em um `HashSet`, sem e com `equals`/`hashCode`, e compare o tamanho do conjunto.

## 6. Java moderno: records, enums, sealed e pattern matching

Do Java 16 ao 21, a linguagem ganhou recursos que reduzem muito o código repetitivo e tornam a modelagem de dados mais segura. Todo código Java novo deveria usá-los.

### 6.1 Records: classes de dados em uma linha

Um **record** é uma classe imutável feita para carregar dados. O compilador gera o construtor, os métodos de acesso, `equals`, `hashCode` e `toString`:

```java
public record Produto(String nome, BigDecimal preco, int estoque) {

    public Produto {                                   // construtor compacto: validação
        Objects.requireNonNull(nome, "nome");
        if (preco.signum() < 0) throw new IllegalArgumentException("preço negativo");
    }

    public boolean disponivel() { return estoque > 0; }   // métodos extras são permitidos

    public Produto comPreco(BigDecimal novo) {           // “alterar” = criar outro
        return new Produto(nome, novo, estoque);
    }
}

var p = new Produto("Teclado", new BigDecimal("199.90"), 10);
IO.println(p.nome());   // métodos de acesso têm o nome do campo, sem "get"
IO.println(p);          // Produto[nome=Teclado, preco=199.90, estoque=10]
```

Use records para DTOs, respostas de API, chaves de mapa, valores como `Cpf` ou `Dinheiro` e resultados de métodos que devolvem mais de um valor.

### 6.2 Enums: conjuntos fechados de valores

```java
public enum StatusPedido {
    CRIADO("Criado"), PAGO("Pago"), ENVIADO("Enviado"), ENTREGUE("Entregue"), CANCELADO("Cancelado");

    private final String rotulo;
    StatusPedido(String rotulo) { this.rotulo = rotulo; }
    public String rotulo() { return rotulo; }

    public boolean podeCancelar() { return this == CRIADO || this == PAGO; }
}

StatusPedido s = StatusPedido.valueOf("PAGO");
for (StatusPedido st : StatusPedido.values()) IO.println(st.rotulo());
```

Enums podem ter atributos, construtores e métodos, e funcionam muito bem com `switch`. Para mapas e conjuntos indexados por enum, use `EnumMap` e `EnumSet`, que são mais rápidos.

### 6.3 Sealed classes: hierarquias fechadas

Uma interface ou classe **sealed** declara exatamente quais tipos podem estendê-la. Assim, o compilador conhece todas as possibilidades:

```java
public sealed interface Forma permits Circulo, Retangulo, Triangulo {}

public record Circulo(double raio) implements Forma {}
public record Retangulo(double largura, double altura) implements Forma {}
public record Triangulo(double base, double altura) implements Forma {}
```

### 6.4 Pattern matching

**instanceof com padrão** testa o tipo e já declara a variável convertida:

```java
if (obj instanceof String s && !s.isBlank()) {
    IO.println(s.toUpperCase());       // s já é String, sem cast
}
```

**switch com padrões e record patterns**: combinado com sealed, o `switch` fica exaustivo e dispensa `default`. Se você criar uma nova `Forma` e esquecer de tratá-la, o código deixa de compilar:

```java
static double area(Forma f) {
    return switch (f) {
        case Circulo(double r) -> Math.PI * r * r;
        case Retangulo(double l, double a) -> l * a;
        case Triangulo(double b, double a) -> b * a / 2;
    };
}

static String classificar(Object o) {
    return switch (o) {
        case null -> "nulo";
        case Integer i when i < 0 -> "inteiro negativo";   // guarda com when
        case Integer i -> "inteiro " + i;
        case String s -> "texto com " + s.length() + " caracteres";
        default -> "outro tipo";
    };
}
```

No Java 22 em diante, use `_` para componentes que não interessam: `case Retangulo(double l, _) -> ...`.

### 6.5 Juntando tudo: modelando com tipos

Records, sealed e switch exaustivo permitem representar regras de negócio como **tipos**, e não como `if` espalhados. Um resultado de pagamento, por exemplo:

```java
sealed interface ResultadoPagamento permits Aprovado, Recusado, EmAnalise {}
record Aprovado(String codigoAutorizacao) implements ResultadoPagamento {}
record Recusado(String motivo) implements ResultadoPagamento {}
record EmAnalise(Duration prazoEstimado) implements ResultadoPagamento {}

String mensagem = switch (resultado) {
    case Aprovado a -> "Pagamento aprovado: " + a.codigoAutorizacao();
    case Recusado r -> "Pagamento recusado: " + r.motivo();
    case EmAnalise e -> "Em análise; resposta em até " + e.prazoEstimado().toMinutes() + " min";
};
```

### Exercícios

- [ ] Reescreva a classe `Cpf` do capítulo 5 como record, com validação de 11 dígitos no construtor compacto.
- [ ] Crie o enum `StatusPedido` com o método `proximo()`, que devolve o status seguinte do fluxo.
- [ ] Modele `Forma` com sealed + records e calcule área e perímetro com `switch` exaustivo. Adicione `Hexagono` e observe o erro do compilador.
- [ ] Escreva um `switch` que receba `Object` e trate `Integer`, `Double`, `String`, `List<?>` e `null`.

## 7. Exceções

Uma **exceção** é um objeto que interrompe o fluxo normal quando algo dá errado. Ela sobe pela pilha de chamadas até encontrar um `catch` que a trate; se ninguém tratar, o programa termina e imprime o *stack trace*.

### 7.1 A hierarquia

| Tipo | Herda de | O compilador obriga a tratar? | Exemplos |
| --- | --- | --- | --- |
| Erros da JVM | `Error` | Não (e não se deve tratar) | `OutOfMemoryError`, `StackOverflowError` |
| Exceções *checked* | `Exception` | Sim: tratar ou declarar com `throws` | `IOException`, `SQLException` |
| Exceções *unchecked* | `RuntimeException` | Não | `NullPointerException`, `IllegalArgumentException`, `IllegalStateException` |

Todas descendem de `Throwable`. **Checked** representam falhas externas esperadas (arquivo inexistente, rede fora). **Unchecked** geralmente indicam bug ou violação de regra (argumento inválido, índice fora do array).

### 7.2 try, catch e finally

```java
try {
    int idade = Integer.parseInt(entrada);
    cadastrar(idade);
} catch (NumberFormatException e) {
    IO.println("Idade inválida: " + entrada);
} catch (IllegalArgumentException | IllegalStateException e) {   // multi-catch
    IO.println("Cadastro recusado: " + e.getMessage());
} finally {
    IO.println("Executa sempre, com ou sem erro");
}
```

Capture do **mais específico** para o mais genérico. Nunca escreva um `catch` vazio: no mínimo, registre o erro em log.

### 7.3 try-with-resources

Recursos que precisam ser fechados (arquivos, conexões, streams) devem ser abertos no cabeçalho do `try`. Eles são fechados automaticamente, mesmo se houver exceção, desde que implementem `AutoCloseable`:

```java
try (var leitor = Files.newBufferedReader(Path.of("dados.csv"))) {
    String linha;
    while ((linha = leitor.readLine()) != null) {
        processar(linha);
    }
} catch (IOException e) {
    throw new UncheckedIOException("Falha ao ler dados.csv", e);
}
```

### 7.4 Lançando e criando exceções

```java
public class SaldoInsuficienteException extends RuntimeException {
    public SaldoInsuficienteException(BigDecimal saldo, BigDecimal valor) {
        super("Saldo de " + saldo + " insuficiente para saque de " + valor);
    }
}

public void sacar(BigDecimal valor) {
    if (valor.compareTo(saldo) > 0) {
        throw new SaldoInsuficienteException(saldo, valor);
    }
    saldo = saldo.subtract(valor);
}
```

Ao converter uma exceção em outra, sempre passe a original como **causa** (`new MinhaException("mensagem", e)`), para não perder o *stack trace*.

### 7.5 Boas práticas

- Valide argumentos cedo (*fail fast*): `Objects.requireNonNull(cliente, "cliente")`.
- Use exceções para situações **excepcionais**, nunca para controle de fluxo normal.
- Mensagens devem dizer o que falhou e com qual valor.
- Prefira exceções *unchecked* de domínio (`PedidoNaoEncontradoException`) em vez de propagar `Exception` genérica.
- Não capture `Exception` ou `Throwable` sem necessidade: você pode esconder bugs.
- Para “valor que pode não existir”, use `Optional` (capítulo 9) em vez de devolver `null` ou lançar exceção.

### 7.6 Lendo um stack trace

```text
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.length()" because "nome" is null
    at br.com.loja.Cliente.validar(Cliente.java:27)
    at br.com.loja.CadastroService.cadastrar(CadastroService.java:14)
    at br.com.loja.Main.main(Main.java:8)
```

Leia de cima para baixo: a primeira linha diz o **tipo** e o **motivo**; a linha seguinte, **onde** aconteceu (`Cliente.java`, linha 27); as demais mostram o caminho das chamadas até ali. As mensagens detalhadas de `NullPointerException` indicam exatamente qual variável era nula.

### Exercícios

- [ ] Crie `SaldoInsuficienteException` e use-a no método `sacar` da sua `ContaBancaria`.
- [ ] Leia um número do teclado repetidamente até o usuário digitar um valor válido, tratando `NumberFormatException`.
- [ ] Leia um arquivo de texto com try-with-resources e mostre uma mensagem amigável se ele não existir.
- [ ] Provoque um `NullPointerException` de propósito e interprete o stack trace.

## 8. Generics e Collections

O **Collections Framework** oferece estruturas de dados prontas e otimizadas. Os **generics** (`<T>`) dizem ao compilador o tipo dos elementos, eliminando casts e erros de tipo em tempo de execução.

### 8.1 Generics

```java
List<String> nomes = new ArrayList<>();   // <> infere o tipo do lado esquerdo
nomes.add("Ana");
// nomes.add(42);                         // erro de compilação

public record Par<A, B>(A primeiro, B segundo) {}     // tipo genérico próprio
var p = new Par<>("idade", 30);

public static <T extends Comparable<T>> T maior(List<T> itens) {   // método genérico com limite
    T max = itens.getFirst();
    for (T item : itens) if (item.compareTo(max) > 0) max = item;
    return max;
}
```

**Curingas**: `List<? extends Number>` aceita listas de `Integer`, `Double` etc. para **leitura**; `List<? super Integer>` aceita listas onde se pode **escrever** `Integer`. Mnemônico: *PECS* — *Producer Extends, Consumer Super*.

### 8.2 As interfaces principais

| Interface | Característica | Implementação padrão | Alternativas |
| --- | --- | --- | --- |
| `List` | Ordenada, aceita repetidos, acesso por índice | `ArrayList` | `LinkedList` (raramente vale a pena) |
| `Set` | Sem repetidos | `HashSet` | `LinkedHashSet` (mantém inserção), `TreeSet` (ordenado) |
| `Map` | Pares chave → valor, chaves únicas | `HashMap` | `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap` |
| `Queue` / `Deque` | Fila (FIFO) e pilha (LIFO) | `ArrayDeque` | `PriorityQueue` (por prioridade) |

Declare a variável pela **interface** (`List<String> x = new ArrayList<>()`): assim você pode trocar a implementação sem mudar o resto do código.

### 8.3 List

```java
List<String> frutas = new ArrayList<>(List.of("uva", "maçã", "banana"));
frutas.add("pera");
frutas.remove("uva");
String primeira = frutas.getFirst();   // Java 21+: getFirst, getLast, reversed
boolean tem = frutas.contains("maçã");
frutas.sort(Comparator.naturalOrder());
frutas.removeIf(f -> f.startsWith("b"));
```

### 8.4 Set e Map

```java
Set<String> tags = new HashSet<>();
tags.add("java");
tags.add("java");            // ignorado: já existe
IO.println(tags.size());     // 1

Map<String, Integer> estoque = new HashMap<>();
estoque.put("teclado", 10);
estoque.put("mouse", 25);
int qtd = estoque.getOrDefault("monitor", 0);
estoque.merge("mouse", 5, Integer::sum);        // soma 5 ao valor existente
estoque.computeIfAbsent("webcam", k -> 0);

for (var entrada : estoque.entrySet()) {
    IO.println(entrada.getKey() + " = " + entrada.getValue());
}
```

### 8.5 Coleções imutáveis

`List.of`, `Set.of` e `Map.of` criam coleções que não podem ser alteradas (lançam `UnsupportedOperationException`). Use-as por padrão para dados fixos e para devolver coleções de métodos sem expor o estado interno. Para copiar uma coleção como imutável: `List.copyOf(lista)`.

### 8.6 Ordenação: Comparable e Comparator

```java
public record Aluno(String nome, double nota) implements Comparable<Aluno> {
    @Override
    public int compareTo(Aluno outro) {           // ordem natural: por nome
        return nome.compareTo(outro.nome);
    }
}

alunos.sort(null);                                 // usa compareTo
alunos.sort(Comparator.comparingDouble(Aluno::nota).reversed()
                      .thenComparing(Aluno::nome));   // por nota desc, depois nome
```

### 8.7 Complexidade das operações mais comuns

| Operação | `ArrayList` | `HashSet` / `HashMap` | `TreeSet` / `TreeMap` |
| --- | --- | --- | --- |
| Adicionar | O(1) amortizado (no fim) | O(1) | O(log n) |
| Buscar por valor/chave | O(n) | O(1) | O(log n) |
| Acessar por índice | O(1) | — | — |
| Remover | O(n) | O(1) | O(log n) |
| Iteração ordenada | ordem de inserção | sem ordem garantida | ordenada |

### Exercícios

- [ ] Leia uma frase e conte a frequência de cada palavra com `Map<String, Integer>` e `merge`.
- [ ] Remova duplicados de uma lista preservando a ordem original (dica: `LinkedHashSet`).
- [ ] Ordene uma lista de `Produto` por preço crescente e, em caso de empate, por nome.
- [ ] Implemente a verificação de parênteses balanceados em uma expressão usando `ArrayDeque` como pilha.

## 9. Programação funcional: lambdas, Streams e Optional

Desde o Java 8, você pode passar **comportamento** como argumento. Lambdas e Streams permitem descrever *o que* fazer com uma coleção (filtrar, transformar, agrupar) em vez de escrever laços passo a passo.

### 9.1 Lambdas e interfaces funcionais

Uma **lambda** é uma função anônima: `(parâmetros) -> expressão`. Ela pode ser usada onde se espera uma **interface funcional**, ou seja, uma interface com um único método abstrato.

| Interface | Método | Recebe → devolve | Exemplo |
| --- | --- | --- | --- |
| `Predicate<T>` | `test` | T → boolean | `s -> s.isBlank()` |
| `Function<T, R>` | `apply` | T → R | `s -> s.length()` |
| `Consumer<T>` | `accept` | T → nada | `s -> IO.println(s)` |
| `Supplier<T>` | `get` | nada → T | `() -> LocalDate.now()` |
| `UnaryOperator<T>` | `apply` | T → T | `s -> s.trim()` |
| `BinaryOperator<T>` | `apply` | (T, T) → T | `(a, b) -> a + b` |

```java
Predicate<String> vazio = String::isBlank;
Function<String, Integer> tamanho = String::length;
Comparator<Produto> porPreco = (a, b) -> a.preco().compareTo(b.preco());
Runnable tarefa = () -> IO.println("executando");
```

**Method references** (`Classe::metodo`) são atalhos para lambdas que apenas chamam um método: `String::toUpperCase`, `IO::println`, `Produto::new`.

### 9.2 Streams: um pipeline em três partes

Um Stream tem **origem** (uma coleção, um array, `Stream.of`), **operações intermediárias** (que devolvem outro Stream e são preguiçosas) e uma **operação terminal**, que dispara o processamento e produz o resultado.

```java
List<Produto> produtos = carregarProdutos();

List<String> nomesBaratos = produtos.stream()            // origem
    .filter(p -> p.preco().compareTo(new BigDecimal("100")) < 0)   // intermediária
    .sorted(Comparator.comparing(Produto::preco))         // intermediária
    .map(Produto::nome)                                   // intermediária
    .toList();                                            // terminal
```

| Intermediárias | Terminais |
| --- | --- |
| `filter`, `map`, `flatMap`, `distinct`, `sorted`, `limit`, `skip`, `peek`, `takeWhile`, `dropWhile`, `mapMulti`, `gather` | `toList`, `collect`, `forEach`, `count`, `sum`, `min`, `max`, `reduce`, `anyMatch`, `allMatch`, `noneMatch`, `findFirst` |

### 9.3 Coletores mais usados

```java
// agrupar por categoria
Map<String, List<Produto>> porCategoria = produtos.stream()
    .collect(Collectors.groupingBy(Produto::categoria));

// contar por categoria
Map<String, Long> contagem = produtos.stream()
    .collect(Collectors.groupingBy(Produto::categoria, Collectors.counting()));

// separar em dois grupos
Map<Boolean, List<Produto>> emEstoque = produtos.stream()
    .collect(Collectors.partitioningBy(Produto::disponivel));

// juntar textos
String lista = produtos.stream().map(Produto::nome).collect(Collectors.joining(", "));

// somar valores
BigDecimal total = produtos.stream()
    .map(Produto::preco)
    .reduce(BigDecimal.ZERO, BigDecimal::add);

// estatísticas de primitivos
IntSummaryStatistics stats = produtos.stream().mapToInt(Produto::estoque).summaryStatistics();
```

> **Atenção:** um Stream só pode ser consumido uma vez, e as lambdas não devem alterar estado externo (por exemplo, adicionar itens a uma lista de fora). Se precisar de efeito colateral, um `for` comum costuma ser mais claro.

### 9.4 Optional: ausência explícita de valor

`Optional<T>` é um contêiner que pode ou não ter um valor. Use-o como **retorno** de métodos que podem não encontrar nada, em vez de devolver `null`:

```java
public Optional<Cliente> buscarPorEmail(String email) { ... }

String nome = buscarPorEmail("ana@exemplo.com")
    .map(Cliente::nome)
    .orElse("Visitante");

Cliente c = buscarPorEmail(email)
    .orElseThrow(() -> new ClienteNaoEncontradoException(email));

buscarPorEmail(email).ifPresentOrElse(
    cli -> enviarBoasVindas(cli),
    () -> IO.println("Cliente não encontrado"));
```

Evite `Optional` em atributos, parâmetros e coleções (use lista vazia, nunca `Optional<List<T>>`). E evite `get()` sem verificar: ele lança exceção se estiver vazio.

### Exercícios

- [ ] A partir de uma lista de `Pedido` (cliente, valor, data), calcule com Streams: o total vendido, o maior pedido e o total por cliente.
- [ ] Liste as cinco palavras mais frequentes de um texto (dica: `groupingBy` + `counting` + ordenação das entradas).
- [ ] Reescreva com Streams um laço seu que filtre e transforme uma lista; compare a legibilidade.
- [ ] Crie `Optional<Produto> buscarPorNome(String)` e use `map`, `filter` e `orElseThrow` sobre o resultado.

## 10. Arquivos e datas

Para arquivos, use a API **`java.nio.file`** (`Path` e `Files`); para datas e horas, a API **`java.time`**. As classes antigas (`File`, `Date`, `Calendar`, `SimpleDateFormat`) ainda existem por compatibilidade, mas não devem ser usadas em código novo.

### 10.1 Lendo e escrevendo arquivos

```java
Path arquivo = Path.of("dados", "clientes.txt");   // dados/clientes.txt

// arquivos pequenos: tudo de uma vez
String conteudo = Files.readString(arquivo);
List<String> linhas = Files.readAllLines(arquivo);
Files.writeString(Path.of("saida.txt"), "Olá, arquivo!\n");
Files.write(Path.of("lista.txt"), List.of("a", "b", "c"),
            StandardOpenOption.CREATE, StandardOpenOption.APPEND);

// arquivos grandes: linha a linha, sem carregar tudo na memória
try (Stream<String> stream = Files.lines(arquivo)) {
    long ativos = stream.filter(l -> l.endsWith(";ATIVO")).count();
}
```

Todas essas leituras usam UTF-8 por padrão (desde o Java 18). Para outro encoding, passe-o explicitamente: `Files.readString(arquivo, StandardCharsets.ISO_8859_1)`.

### 10.2 Operações com arquivos e diretórios

```java
Files.exists(arquivo);
Files.createDirectories(Path.of("relatorios/2026"));
Files.copy(origem, destino, StandardCopyOption.REPLACE_EXISTING);
Files.move(origem, destino);
Files.deleteIfExists(temporario);
long bytes = Files.size(arquivo);

try (Stream<Path> itens = Files.walk(Path.of("src"))) {      // percorre recursivamente
    itens.filter(p -> p.toString().endsWith(".java")).forEach(IO::println);
}
```

### 10.3 Exemplo: processando um CSV

```java
record Venda(String produto, int quantidade, BigDecimal valor) {}

List<Venda> vendas;
try (Stream<String> linhas = Files.lines(Path.of("vendas.csv"))) {
    vendas = linhas.skip(1)                                   // pula o cabeçalho
        .map(l -> l.split(";"))
        .map(c -> new Venda(c[0], Integer.parseInt(c[1]), new BigDecimal(c[2])))
        .toList();
}
BigDecimal faturamento = vendas.stream()
    .map(v -> v.valor().multiply(BigDecimal.valueOf(v.quantidade())))
    .reduce(BigDecimal.ZERO, BigDecimal::add);
```

Para CSVs reais, com aspas e separadores dentro dos campos, use uma biblioteca como Apache Commons CSV ou OpenCSV.

### 10.4 Datas e horas com java.time

| Classe | Representa | Exemplo |
| --- | --- | --- |
| `LocalDate` | Data sem hora nem fuso | aniversário, vencimento |
| `LocalTime` | Hora sem data | horário de abertura |
| `LocalDateTime` | Data e hora sem fuso | agendamento local |
| `ZonedDateTime` | Data e hora com fuso | reunião entre São Paulo e Lisboa |
| `Instant` | Ponto exato na linha do tempo (UTC) | registro de evento, auditoria |
| `Duration` | Quantidade de tempo em horas, minutos, segundos | tempo de resposta |
| `Period` | Quantidade de tempo em anos, meses, dias | idade, prazo contratual |

Todas são **imutáveis** e seguras para uso entre threads.

```java
LocalDate hoje = LocalDate.now();
LocalDate vencimento = hoje.plusDays(30);
boolean atrasado = hoje.isAfter(vencimento);
int idade = Period.between(LocalDate.of(1990, 5, 17), hoje).getYears();
long dias = ChronoUnit.DAYS.between(hoje, vencimento);
DayOfWeek dia = hoje.getDayOfWeek();

ZonedDateTime emSp = ZonedDateTime.now(ZoneId.of("America/Sao_Paulo"));
ZonedDateTime emLisboa = emSp.withZoneSameInstant(ZoneId.of("Europe/Lisbon"));

var formato = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm");
String texto = LocalDateTime.now().format(formato);
LocalDate data = LocalDate.parse("25/12/2026", DateTimeFormatter.ofPattern("dd/MM/yyyy"));
```

> **Dica:** em sistemas, armazene momentos como `Instant` (UTC) e converta para o fuso do usuário apenas na exibição. Isso evita erros com horário de verão e usuários em fusos diferentes.

### Exercícios

- [ ] Leia um CSV de vendas e gere um arquivo `resumo.txt` com o total por produto.
- [ ] Liste todos os arquivos `.log` de um diretório, com tamanho e data de modificação, ordenados do maior para o menor.
- [ ] Calcule quantos dias úteis (segunda a sexta) existem entre duas datas.
- [ ] Mostre a hora atual em São Paulo, Nova York e Tóquio no formato `dd/MM/yyyy HH:mm`.

## 11. Concorrência

Concorrência é executar várias tarefas ao mesmo tempo, para aproveitar múltiplos núcleos ou para não ficar parado esperando rede e disco. Em Java moderno, a regra é: **não crie threads à mão**; use executores e, para tarefas que esperam I/O, **virtual threads**.

### 11.1 Threads e Runnable

```java
Thread t = Thread.ofPlatform().start(() -> IO.println("rodando em " + Thread.currentThread()));
t.join();   // espera terminar
```

Uma **thread de plataforma** corresponde a uma thread do sistema operacional: é pesada (cerca de 1 MB de pilha) e limitada a alguns milhares por processo.

### 11.2 ExecutorService

Um executor gerencia um conjunto de threads e uma fila de tarefas. Você envia tarefas e recebe um `Future` com o resultado:

```java
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {   // AutoCloseable desde o Java 19
    Future<Integer> f = pool.submit(() -> calcularAlgoPesado());
    Integer resultado = f.get();   // bloqueia até o resultado ficar pronto
}                                  // close() aguarda as tarefas terminarem
```

Use pools de tamanho fixo (próximo ao número de núcleos, `Runtime.getRuntime().availableProcessors()`) para tarefas que gastam **CPU**.

### 11.3 Virtual threads (Java 21+)

**Virtual threads** são threads leves gerenciadas pela JVM. Você pode criar milhões delas: quando uma bloqueia esperando I/O (uma chamada HTTP, uma consulta ao banco), a JVM a desmonta e usa a thread do sistema para outra tarefa. Isso permite escrever código simples e bloqueante com a escalabilidade de código assíncrono.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> respostas = urls.stream()
        .map(url -> executor.submit(() -> baixar(url)))   // uma virtual thread por tarefa
        .toList();
    for (var r : respostas) IO.println(r.get().length());
}
```

| Tipo de tarefa | Use |
| --- | --- |
| Muita espera de I/O (APIs, banco, arquivos) | Virtual threads |
| Cálculo intenso de CPU | Pool fixo de threads de plataforma, ou `parallelStream` |

Não faça pool de virtual threads: crie uma por tarefa. Para limitar o acesso a um recurso escasso (por exemplo, no máximo 10 conexões simultâneas), use um `Semaphore`.

### 11.4 CompletableFuture: compondo tarefas assíncronas

```java
CompletableFuture<Cliente> cliente = CompletableFuture.supplyAsync(() -> buscarCliente(id));
CompletableFuture<List<Pedido>> pedidos = CompletableFuture.supplyAsync(() -> buscarPedidos(id));

CompletableFuture<Resumo> resumo = cliente
    .thenCombine(pedidos, Resumo::new)          // combina os dois resultados
    .orTimeout(2, TimeUnit.SECONDS)
    .exceptionally(e -> Resumo.vazio());        // trata falhas

Resumo r = resumo.join();
```

### 11.5 Condições de corrida e sincronização

Quando várias threads alteram o mesmo dado, o resultado pode ficar errado. `contador++` parece uma operação única, mas são três (ler, somar, gravar), e duas threads podem se intercalar.

```java
// Errado: perde incrementos
int contador = 0;

// Certo: operações atômicas
AtomicInteger contador = new AtomicInteger();
contador.incrementAndGet();

// Certo: bloqueio explícito
private final ReentrantLock lock = new ReentrantLock();
void transferir() {
    lock.lock();
    try { /* seção crítica */ } finally { lock.unlock(); }
}
```

Estratégias, da mais segura para a mais trabalhosa:

1. **Imutabilidade**: records e coleções imutáveis não precisam de sincronização.
2. **Não compartilhar**: cada tarefa trabalha com seus próprios dados e devolve um resultado.
3. **Estruturas concorrentes prontas**: `ConcurrentHashMap`, `AtomicLong`, `LinkedBlockingQueue`.
4. **Locks**: `synchronized` ou `ReentrantLock`, mantendo a seção crítica a menor possível.

> **Para conhecer:** o Java 25 finalizou os **scoped values** (`ScopedValue`), uma alternativa mais segura ao `ThreadLocal` para compartilhar dados de contexto (como o usuário da requisição) com virtual threads. A **structured concurrency** (`StructuredTaskScope`), que trata um grupo de subtarefas como uma unidade, ainda está em *preview*.

### Exercícios

- [ ] Some os números de 1 a 100 milhões dividindo o trabalho em 4 tarefas de um pool fixo e compare o tempo com a versão sequencial.
- [ ] Simule 10.000 chamadas que fazem `Thread.sleep(Duration.ofMillis(100))` com virtual threads e, depois, com um pool de 100 threads de plataforma. Compare os tempos.
- [ ] Demonstre uma condição de corrida com um `int` compartilhado e corrija com `AtomicInteger`.
- [ ] Combine duas consultas simuladas com `CompletableFuture` e aplique um timeout de 1 segundo.

## 12. Build e organização de projetos: Maven e Gradle

Projetos reais não são compilados com `javac` à mão. Uma **ferramenta de build** baixa dependências, compila, roda testes e empacota o resultado em um `.jar`, sempre do mesmo jeito, na sua máquina e no servidor de CI. As duas dominantes são **Maven** e **Gradle**.

### 12.1 Pacotes e estrutura de diretórios

Classes são organizadas em **pacotes**, que correspondem a pastas. Por convenção, o nome começa pelo domínio invertido da organização:

```text
loja/
├── pom.xml                      (ou build.gradle.kts)
└── src/
    ├── main/
    │   ├── java/br/com/exemplo/loja/
    │   │   ├── App.java
    │   │   ├── dominio/Produto.java
    │   │   └── servico/CarrinhoService.java
    │   └── resources/           (arquivos de configuração, templates)
    └── test/
        └── java/br/com/exemplo/loja/
            └── servico/CarrinhoServiceTest.java
```

```java
package br.com.exemplo.loja.servico;       // primeira linha do arquivo

import br.com.exemplo.loja.dominio.Produto;
import java.util.List;
```

Maven e Gradle usam essa mesma estrutura; a IDE reconhece tudo automaticamente.

### 12.2 Maven

O Maven descreve o projeto em um arquivo XML, o `pom.xml`. Cada dependência é identificada por **groupId**, **artifactId** e **version**, e é baixada do repositório Maven Central.

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>br.com.exemplo</groupId>
  <artifactId>loja</artifactId>
  <version>1.0.0-SNAPSHOT</version>

  <properties>
    <maven.compiler.release>25</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.junit</groupId>
        <artifactId>junit-bom</artifactId>
        <version>6.1.2</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.assertj</groupId>
      <artifactId>assertj-core</artifactId>
      <version>3.27.3</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.5.2</version>
      </plugin>
    </plugins>
  </build>
</project>
```

O **BOM** (*bill of materials*) fixa as versões de uma família de bibliotecas; por isso `junit-jupiter` não precisa de `<version>`. Os números de versão aqui são exemplos: confira as mais recentes em [central.sonatype.com](https://central.sonatype.com).

| Comando | O que faz |
| --- | --- |
| `mvn compile` | Compila o código de `src/main` |
| `mvn test` | Compila e roda os testes |
| `mvn package` | Gera o `.jar` em `target/` |
| `mvn verify` | Roda tudo, inclusive verificações de qualidade configuradas |
| `mvn clean` | Apaga a pasta `target/` |
| `mvn dependency:tree` | Mostra a árvore de dependências, útil para conflitos de versão |

Os **escopos** controlam onde a dependência está disponível: `compile` (padrão, em tudo), `test` (só nos testes), `runtime` (só na execução, como drivers JDBC) e `provided` (fornecida pelo ambiente).

### 12.3 Gradle

O Gradle descreve o build em código (Kotlin ou Groovy). É mais flexível e costuma ser mais rápido em projetos grandes, graças ao cache de build. É o padrão no Android.

```kotlin
// build.gradle.kts
plugins {
    application
}

java {
    toolchain { languageVersion = JavaLanguageVersion.of(25) }
}

repositories { mavenCentral() }

dependencies {
    testImplementation(platform("org.junit:junit-bom:6.1.2"))
    testImplementation("org.junit.jupiter:junit-jupiter")
    testRuntimeOnly("org.junit.platform:junit-platform-launcher")
    testImplementation("org.assertj:assertj-core:3.27.3")
}

application { mainClass = "br.com.exemplo.loja.App" }

tasks.test { useJUnitPlatform() }
```

```bash
./gradlew build     # compila, testa e empacota
./gradlew test
./gradlew run       # executa a mainClass
```

|  | Maven | Gradle |
| --- | --- | --- |
| Configuração | XML declarativo (`pom.xml`) | Código Kotlin ou Groovy |
| Curva de aprendizado | Menor; tudo segue a convenção | Maior; mais poder de customização |
| Desempenho | Bom | Melhor em projetos grandes (build incremental e cache) |
| Onde predomina | Back-end corporativo | Android, projetos grandes e multimódulos |

> **Dica:** use sempre o **wrapper** (`mvnw` ou `gradlew`), gerado junto com o projeto e versionado no Git. Ele baixa a versão exata da ferramenta, e todos da equipe e o CI usam a mesma.

### 12.4 Módulos da plataforma (JPMS)

Desde o Java 9, um projeto pode declarar um `module-info.java` com quais pacotes exporta e de quais módulos depende. Isso fortalece o encapsulamento e permite gerar runtimes enxutos com `jlink`. A maioria das aplicações de negócio, inclusive as feitas com Spring Boot, ainda não usa módulos; aprenda o conceito, mas não é pré-requisito para começar.

```java
module br.com.exemplo.loja {
    requires java.net.http;
    exports br.com.exemplo.loja.api;
}
```

### Exercícios

- [ ] Crie um projeto Maven pela IDE (ou com `mvn archetype:generate`), mova para ele as classes dos capítulos anteriores e organize-as em pacotes.
- [ ] Rode `mvn package` e execute o `.jar` gerado com `java -cp target/loja-1.0.0-SNAPSHOT.jar br.com.exemplo.loja.App`.
- [ ] Adicione uma biblioteca externa (por exemplo, Jackson, para JSON) e veja a árvore com `mvn dependency:tree`.
- [ ] Recrie o mesmo projeto com Gradle e compare os tempos de um segundo build.

## 13. Testes automatizados

Testes automatizados são código que verifica código. Eles dão segurança para mudar o sistema: se algo quebrar, você descobre em segundos, e não em produção. O trio padrão em Java é **JUnit** (execução), **AssertJ** (verificações legíveis) e **Mockito** (dublês de dependências).

A versão atual é o **JUnit 6** (lançado em setembro de 2025, exige Java 17 ou superior). A API de testes, chamada **Jupiter**, é a mesma do JUnit 5: o que você ler sobre JUnit 5 continua valendo ([notas de versão](https://docs.junit.org/6.0.0/release-notes/)).

### 13.1 O primeiro teste

```java
class CarrinhoTest {

    private Carrinho carrinho;

    @BeforeEach
    void prepara() {                     // roda antes de cada teste
        carrinho = new Carrinho();
    }

    @Test
    void totalDeCarrinhoVazioEZero() {
        assertThat(carrinho.total()).isEqualByComparingTo("0");
    }

    @Test
    void somaPrecoVezesQuantidade() {
        // Arrange (prepara)
        carrinho.adicionar(new Produto("Caneta", new BigDecimal("2.50"), 100), 4);
        // Act (executa)
        BigDecimal total = carrinho.total();
        // Assert (verifica)
        assertThat(total).isEqualByComparingTo("10.00");
    }

    @Test
    void recusaQuantidadeNegativa() {
        var produto = new Produto("Caneta", new BigDecimal("2.50"), 100);
        assertThatThrownBy(() -> carrinho.adicionar(produto, -1))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("quantidade");
    }
}
```

Rode com `mvn test`, `./gradlew test` ou pelo botão verde da IDE. Estruture cada teste em **Arrange, Act, Assert** e dê nomes que descrevam o comportamento esperado.

### 13.2 Anotações principais

| Anotação | Uso |
| --- | --- |
| `@Test` | Marca um método de teste |
| `@BeforeEach` / `@AfterEach` | Roda antes e depois de cada teste |
| `@BeforeAll` / `@AfterAll` | Roda uma vez antes e depois de todos (método `static`) |
| `@DisplayName("...")` | Nome legível no relatório |
| `@Nested` | Agrupa testes relacionados em uma classe interna |
| `@ParameterizedTest` | Roda o mesmo teste com várias entradas |
| `@Disabled` | Desativa temporariamente, com motivo |
| `@Tag("lento")` | Classifica testes para rodar subconjuntos |

### 13.3 Testes parametrizados

```java
@ParameterizedTest
@CsvSource({
    "100.00, 0,  100.00",
    "100.00, 10, 90.00",
    "80.00,  25, 60.00"
})
void aplicaDesconto(BigDecimal preco, int percentual, BigDecimal esperado) {
    assertThat(Desconto.aplicar(preco, percentual)).isEqualByComparingTo(esperado);
}
```

### 13.4 Mockito: isolando dependências

Um teste de unidade testa **uma** classe. Se ela depende de um banco, de um e-mail ou de uma API externa, substitua essas dependências por **mocks**: objetos falsos cujo comportamento você controla e cujas chamadas você verifica.

```java
@ExtendWith(MockitoExtension.class)
class PedidoServiceTest {

    @Mock PedidoRepository repositorio;
    @Mock Notificador notificador;
    @InjectMocks PedidoService servico;     // recebe os mocks no construtor

    @Test
    void confirmaPedidoENotificaCliente() {
        var pedido = new Pedido(42L, "ana@exemplo.com", StatusPedido.CRIADO);
        when(repositorio.buscar(42L)).thenReturn(Optional.of(pedido));

        servico.confirmar(42L);

        verify(repositorio).salvar(argThat(p -> p.status() == StatusPedido.PAGO));
        verify(notificador).enviar(eq("ana@exemplo.com"), anyString());
    }

    @Test
    void falhaQuandoPedidoNaoExiste() {
        when(repositorio.buscar(99L)).thenReturn(Optional.empty());
        assertThatThrownBy(() -> servico.confirmar(99L))
            .isInstanceOf(PedidoNaoEncontradoException.class);
        verifyNoInteractions(notificador);
    }
}
```

Dependências Maven: `org.mockito:mockito-junit-jupiter` (escopo `test`), que já traz o `mockito-core`.

### 13.5 A pirâmide de testes e o TDD

| Nível | O que testa | Velocidade | Quantidade |
| --- | --- | --- | --- |
| Unidade | Uma classe isolada, com mocks | Milissegundos | Muitos |
| Integração | Várias partes juntas, com banco real (por exemplo, via Testcontainers) | Segundos | Alguns |
| Ponta a ponta | O sistema inteiro, como o usuário usa | Minutos | Poucos |

No **TDD** (*Test-Driven Development*), você escreve o teste antes do código, em ciclos curtos: **vermelho** (o teste falha), **verde** (o código mínimo para passar) e **refatoração** (melhora o código mantendo os testes verdes). Mesmo sem seguir TDD à risca, escrever o teste junto com a funcionalidade melhora o design: código difícil de testar costuma estar acoplado demais.

Para medir o quanto do código os testes exercitam, use o **JaCoCo**. Trate a cobertura como indicador, não como meta: 100% de cobertura com verificações fracas não garante nada.

### Exercícios

- [ ] Escreva testes para `ContaBancaria`: depósito, saque, saque acima do saldo e valores inválidos.
- [ ] Transforme os testes de desconto em um `@ParameterizedTest` com pelo menos cinco casos, incluindo os limites.
- [ ] Crie um `PedidoService` que use um repositório e um notificador e teste-o com Mockito.
- [ ] Implemente com TDD uma função que valide CPF (dígitos verificadores), escrevendo cada teste antes do código.

## 14. Por dentro da JVM

Você não precisa conhecer a JVM para escrever Java, mas precisa conhecê-la para diagnosticar lentidão, consumo de memória e travamentos em produção. Os pontos centrais são onde os dados ficam, como a memória é liberada e como o código fica rápido.

### 14.1 As áreas de memória

&#91;embedded content: Arquitetura da JVM · carregamento, memória e execução\]

- **Pilha (stack):** cada thread tem a sua. Guarda variáveis locais, parâmetros e referências, organizados em *frames*, um por chamada de método. É liberada automaticamente quando o método termina. Recursão sem fim provoca `StackOverflowError`.
- **Heap:** compartilhado por todas as threads; é onde vivem todos os objetos criados com `new`. Dividido em geração **jovem** (objetos recém-criados; a maioria morre ali) e **velha** (objetos que sobreviveram a várias coletas). Se encher, ocorre `OutOfMemoryError: Java heap space`.
- **Metaspace:** metadados das classes carregadas, fora do heap.

```java
void exemplo() {
    int x = 10;                         // x fica na pilha
    Produto p = new Produto(...);       // a referência p fica na pilha; o objeto, no heap
}                                       // ao sair, o frame some; o objeto vira lixo se ninguém mais o referenciar
```

### 14.2 Garbage collection

O **garbage collector** (GC) encontra objetos que não são mais alcançáveis a partir das referências vivas e libera a memória deles. Você não chama `free` nem `delete`. Porém, ainda é possível ter **vazamento de memória**: objetos que continuam referenciados sem necessidade (por exemplo, em um `Map` estático usado como cache que nunca é limpo).

| Coletor | Opção | Indicado para |
| --- | --- | --- |
| G1 | padrão (`-XX:+UseG1GC`) | Uso geral; bom equilíbrio entre vazão e pausas |
| ZGC | `-XX:+UseZGC` | Heaps grandes e baixa latência, com pausas abaixo de 1 ms |
| Shenandoah | `-XX:+UseShenandoahGC` | Baixa latência (disponível em builds como Temurin e Red Hat) |
| Parallel | `-XX:+UseParallelGC` | Processamento em lote, priorizando vazão |
| Serial | `-XX:+UseSerialGC` | Aplicações pequenas, um núcleo, contêineres mínimos |

### 14.3 JIT: por que o Java fica mais rápido com o tempo

A JVM começa **interpretando** o bytecode. Ela mede quais métodos são executados com mais frequência (*hot spots*) e os compila para código nativo: primeiro com o compilador **C1**, rápido de compilar, e depois com o **C2**, que aplica otimizações agressivas como *inlining* e eliminação de código morto. Por isso uma aplicação Java costuma ficar mais rápida alguns segundos ou minutos depois de iniciar (o *warm-up*), e por isso medições de desempenho exigem ferramentas como o **JMH**.

Para inicialização instantânea (funções serverless, CLIs), existe a compilação antecipada com **GraalVM Native Image**, que gera um executável nativo.

### 14.4 Opções essenciais da JVM

```bash
java -Xms512m -Xmx2g -jar app.jar                     # heap inicial e máximo
java -XX:MaxRAMPercentage=75 -jar app.jar            # em contêineres: % da memória do contêiner
java -XX:+UseZGC -jar app.jar                         # escolhe o coletor
java -Xlog:gc -jar app.jar                            # registra as coletas no log
java -XX:+HeapDumpOnOutOfMemoryError -jar app.jar     # gera dump do heap ao estourar a memória
```

Em contêineres (Docker, Kubernetes), a JVM detecta os limites de CPU e memória. Prefira `MaxRAMPercentage` a um `-Xmx` fixo, deixando espaço para memória fora do heap (threads, metaspace, buffers).

### 14.5 Ferramentas de diagnóstico

| Ferramenta | Uso |
| --- | --- |
| `jcmd <pid> help` | Canivete suíço: lista comandos disponíveis para um processo |
| `jcmd <pid> Thread.print` | Mostra o que cada thread está fazendo (detecta *deadlocks*) |
| `jcmd <pid> GC.heap_info` | Ocupação atual do heap |
| `jcmd <pid> JFR.start duration=60s filename=app.jfr` | Grava um perfil com o Java Flight Recorder, com impacto mínimo |
| JDK Mission Control | Abre e analisa gravações `.jfr` |
| VisualVM | Monitoramento visual de CPU, memória e threads |
| Eclipse MAT | Analisa *heap dumps* para encontrar vazamentos |

Descubra o `pid` com `jps -l` ou `jcmd` sem argumentos.

### Exercícios

- [ ] Escreva um método recursivo sem caso base e observe o `StackOverflowError`.
- [ ] Crie um programa que adicione objetos a uma lista estática em um laço infinito; rode com `-Xmx64m -XX:+HeapDumpOnOutOfMemoryError` e abra o dump no Eclipse MAT ou no VisualVM.
- [ ] Rode o mesmo programa com `-Xlog:gc`, primeiro com G1 e depois com ZGC, e compare os registros.
- [ ] Grave 30 segundos de uma aplicação sua com JFR e identifique, no Mission Control, os métodos que mais consomem CPU.

## 15. Ecossistema: JDBC, JPA e Spring Boot

Na prática profissional, Java é usado principalmente para construir **back-ends**: APIs que recebem requisições, aplicam regras de negócio e guardam dados em um banco. Este capítulo mostra as três camadas dessa pilha, da mais baixa para a mais produtiva.

### 15.1 JDBC: acesso direto ao banco

O **JDBC** é a API padrão para executar SQL. Todo framework de persistência usa JDBC por baixo.

```java
String url = "jdbc:postgresql://localhost:5432/loja";
String sql = "SELECT id, nome, preco FROM produto WHERE preco < ?";

try (Connection conn = DriverManager.getConnection(url, "loja", senha);
     PreparedStatement ps = conn.prepareStatement(sql)) {
    ps.setBigDecimal(1, new BigDecimal("100"));
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) {
            IO.println(rs.getLong("id") + " - " + rs.getString("nome"));
        }
    }
}
```

> **Atenção:** nunca monte SQL concatenando valores do usuário (`"... WHERE nome = '" + nome + "'"`). Isso abre a porta para **SQL injection**. Use sempre `PreparedStatement` com parâmetros `?`.

Em aplicações reais, as conexões vêm de um **pool** (o HikariCP é o padrão), porque abrir uma conexão nova a cada consulta é caro.

### 15.2 JPA e Hibernate: objetos no lugar de SQL

A **JPA** (Jakarta Persistence) mapeia classes para tabelas. O **Hibernate** é a implementação mais usada. Você trabalha com objetos, e o framework gera o SQL:

```java
@Entity
@Table(name = "produto")
public class Produto {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 120)
    private String nome;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal preco;

    @ManyToOne(fetch = FetchType.LAZY)
    private Categoria categoria;

    protected Produto() {}          // exigido pela JPA
    // construtor, getters e métodos de negócio...
}
```

Entidades JPA precisam de um construtor sem argumentos e de estado mutável, então **não** podem ser records. Use records nos DTOs, que são os objetos que entram e saem da API.

Cuidado com o problema **N+1**: carregar uma lista de 100 pedidos e, para cada um, buscar o cliente gera 101 consultas. Resolva com `JOIN FETCH` na consulta ou com `@EntityGraph`.

### 15.3 Spring Boot

O **Spring** é o framework mais usado do ecossistema Java, e o **Spring Boot** é a forma de criar aplicações Spring com configuração mínima: servidor web embutido, configuração automática e um `.jar` executável. A versão atual é a série **4.x**; a 4.1 saiu em junho de 2026 ([Spring Boot 4.1.0](https://spring.io/blog/2026/06/10/spring-boot-4/)).

A base do Spring é a **injeção de dependências**: em vez de uma classe criar suas dependências com `new`, ela as recebe no construtor, e o Spring monta e conecta os objetos (os *beans*). Isso deixa o código desacoplado e fácil de testar com mocks.

Crie o projeto em [start.spring.io](https://start.spring.io) com Java 25 e as dependências **Spring Web**, **Spring Data JPA**, **Validation** e **H2 Database** (banco em memória para desenvolvimento).

### 15.4 Uma API REST completa

**Repositório** — o Spring Data gera a implementação a partir da interface:

```java
public interface ProdutoRepository extends JpaRepository<Produto, Long> {
    List<Produto> findByNomeContainingIgnoreCase(String trecho);   // consulta derivada do nome do método
}
```

**DTOs** — records com validação declarativa:

```java
public record NovoProdutoRequest(
    @NotBlank @Size(max = 120) String nome,
    @NotNull @DecimalMin("0.01") BigDecimal preco) {}

public record ProdutoResponse(Long id, String nome, BigDecimal preco) {
    static ProdutoResponse de(Produto p) {
        return new ProdutoResponse(p.getId(), p.getNome(), p.getPreco());
    }
}
```

**Serviço** — regras de negócio e transações:

```java
@Service
public class ProdutoService {
    private final ProdutoRepository repositorio;

    public ProdutoService(ProdutoRepository repositorio) {   // injeção pelo construtor
        this.repositorio = repositorio;
    }

    @Transactional
    public Produto criar(NovoProdutoRequest req) {
        return repositorio.save(new Produto(req.nome(), req.preco()));
    }

    @Transactional(readOnly = true)
    public Produto buscar(Long id) {
        return repositorio.findById(id)
            .orElseThrow(() -> new ProdutoNaoEncontradoException(id));
    }
}
```

**Controller** — os endpoints HTTP:

```java
@RestController
@RequestMapping("/api/produtos")
public class ProdutoController {
    private final ProdutoService servico;

    public ProdutoController(ProdutoService servico) { this.servico = servico; }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProdutoResponse criar(@Valid @RequestBody NovoProdutoRequest req) {
        return ProdutoResponse.de(servico.criar(req));
    }

    @GetMapping("/{id}")
    public ProdutoResponse buscar(@PathVariable Long id) {
        return ProdutoResponse.de(servico.buscar(id));
    }
}

@RestControllerAdvice
class TratadorDeErros {
    @ExceptionHandler(ProdutoNaoEncontradoException.class)
    ProblemDetail naoEncontrado(ProdutoNaoEncontradoException e) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, e.getMessage());
    }
}
```

**Configuração** (`src/main/resources/application.properties`):

```properties
spring.datasource.url=jdbc:h2:mem:loja
spring.jpa.hibernate.ddl-auto=update
spring.threads.virtual.enabled=true
```

Rode com `./mvnw spring-boot:run` e teste:

```bash
curl -X POST localhost:8080/api/produtos \
     -H "Content-Type: application/json" \
     -d '{"nome": "Teclado", "preco": 199.90}'

curl localhost:8080/api/produtos/1
```

A divisão em **controller → serviço → repositório** é o padrão de mercado: o controller só traduz HTTP, o serviço concentra as regras e o repositório cuida da persistência.

### 15.5 Outros nomes do ecossistema

| Ferramenta | Para que serve |
| --- | --- |
| Quarkus, Micronaut | Frameworks alternativos ao Spring, focados em inicialização rápida e baixo consumo |
| Flyway, Liquibase | Versionamento do esquema do banco (migrações) |
| Jackson | Conversão entre objetos e JSON (usado pelo Spring) |
| Lombok | Gera getters, construtores e builders por anotações; records reduziram sua necessidade |
| SLF4J + Logback | Logs |
| Testcontainers | Sobe bancos e serviços reais em Docker durante os testes |
| Spring Security | Autenticação e autorização (JWT, OAuth2) |
| Kafka, RabbitMQ | Mensageria entre serviços |

### Exercícios

- [ ] Escreva com JDBC um programa que insira e liste produtos em um banco H2 em arquivo.
- [ ] Crie o projeto Spring Boot da seção 15.4 e acrescente os endpoints `GET /api/produtos` (com busca por nome), `PUT /api/produtos/{id}` e `DELETE /api/produtos/{id}`.
- [ ] Envie um produto com preço negativo e observe a resposta de validação.
- [ ] Escreva um teste de integração com `@SpringBootTest` e `MockMvc` para o endpoint de criação.

## 16. Projeto final, boas práticas e trilha de estudos

O projeto final junta todos os capítulos em uma aplicação que você pode colocar no portfólio: uma **API de pedidos para uma loja virtual**, construída em etapas que crescem em dificuldade.

### 16.1 Projeto: API de pedidos

| Etapa | Entrega | Capítulos |
| --- | --- | --- |
| 1. Domínio | Records `Produto`, `ItemPedido` e `Cliente`; enum `StatusPedido` com transições válidas; classe `Pedido` que calcula total com `BigDecimal` | 3 a 6 |
| 2. Regras e erros | Exceções de domínio (`EstoqueInsuficienteException`, `TransicaoInvalidaException`); validações nos construtores | 7 |
| 3. Repositório em memória | `Map<Long, Pedido>`; relatórios com Streams: faturamento por dia, produtos mais vendidos, ticket médio por cliente | 8 e 9 |
| 4. Importação | Ler um catálogo de produtos de CSV e exportar o relatório diário para arquivo | 10 |
| 5. Projeto Maven e testes | Estrutura em pacotes; testes de unidade com JUnit, AssertJ e Mockito, com pelo menos 80% de cobertura nas regras | 12 e 13 |
| 6. API com Spring Boot | Endpoints para criar pedido, consultar, pagar e cancelar; JPA com PostgreSQL; migrações com Flyway; erros como `ProblemDetail` | 15 |
| 7. Concorrência | Consulta paralela de frete em três transportadoras simuladas com virtual threads, usando a resposta mais barata dentro de 2 segundos | 11 |
| 8. Entrega | `Dockerfile`, `docker compose` com o banco, pipeline de CI que roda `mvn verify`, README com instruções | 14 |

Critérios de pronto: todos os testes passam no CI, nenhum valor monetário em `double`, nenhuma regra de negócio no controller e um README que permita a outra pessoa rodar o projeto em menos de cinco minutos.

### 16.2 Boas práticas de código

- **Nomes claros** valem mais que comentários: `calcularFreteExpresso()` dispensa explicação; `calc2()` não.
- **Métodos pequenos**, com uma responsabilidade cada; classes coesas.
- **Imutabilidade por padrão**: records, `final`, `List.of`. Estado mutável só onde for necessário.
- **Programe para interfaces** (`List`, `Repository`), não para implementações.
- **Composição em vez de herança**.
- **Nunca devolva `null`** para coleções (devolva vazia) nem para valores opcionais (use `Optional`).
- **Valide na entrada** e falhe cedo, com mensagens que digam o que e qual valor.
- **Feche recursos** com try-with-resources.
- **Injeção pelo construtor** e dependências `final`.
- **Teste o comportamento**, não a implementação.
- Configure um formatador e um analisador estático (Checkstyle, SpotBugs, SonarQube ou Error Prone) no build, para que as regras não dependam de revisão manual.

### 16.3 Erros comuns de iniciantes

| Erro | Correto |
| --- | --- |
| Comparar Strings com `==` | `a.equals(b)` |
| `double` para dinheiro | `BigDecimal` criado a partir de String |
| `catch (Exception e) {}` vazio | Tratar, relançar com causa ou ao menos registrar em log |
| Sobrescrever `equals` sem `hashCode` | Os dois juntos, ou usar record |
| Concatenar String em laço | `StringBuilder` ou `String.join` |
| `Optional.get()` sem verificar | `orElse`, `orElseThrow`, `map` |
| SQL concatenado com dados do usuário | `PreparedStatement` ou JPA com parâmetros |
| `Date` e `SimpleDateFormat` | `java.time` |
| Modificar uma lista enquanto a percorre com for-each | `removeIf` ou um `Iterator` |

### 16.4 Trilha de estudos depois deste guia

1. **Consolide a linguagem:** leia *Effective Java*, de Joshua Bloch, e resolva exercícios em plataformas como Exercism ou LeetCode.
2. **Aprofunde o Spring:** Spring Security, Spring Data, mensageria com Kafka, observabilidade com Micrometer e OpenTelemetry.
3. **Arquitetura:** princípios SOLID, Clean Architecture, arquitetura hexagonal, Domain-Driven Design e padrões de projeto.
4. **Banco de dados:** SQL avançado, índices, transações e níveis de isolamento.
5. **Nuvem e contêineres:** Docker, Kubernetes e um provedor de nuvem (AWS, Google Cloud ou Azure).
6. **Desempenho:** JFR, JMH, ajuste de GC e GraalVM Native Image.
7. **Certificação (opcional):** Oracle Certified Professional, Java SE Developer. Confira no site da Oracle qual versão do Java o exame atual cobre.

Acompanhe a evolução da linguagem em [openjdk.org](https://openjdk.org) (as JEPs descrevem cada novo recurso) e em [dev.java](https://dev.java), o portal oficial de aprendizado.

## 17. Apêndices

### A. Guia rápido de sintaxe

```java
// Variáveis e tipos
int n = 10;  long l = 10L;  double d = 1.5;  boolean ok = true;  char c = 'x';
var nome = "Ana";                       final int MAX = 100;

// Condicionais e laços
if (a > b) { } else if (a == b) { } else { }
String r = switch (x) { case 1 -> "um"; case 2 -> "dois"; default -> "outro"; };
for (int i = 0; i < n; i++) { }         for (var item : lista) { }
while (cond) { }                        do { } while (cond);

// Classes, records, enums, interfaces
class Pessoa { private final String nome; Pessoa(String nome) { this.nome = nome; } }
record Ponto(int x, int y) {}
enum Cor { VERMELHO, VERDE, AZUL }
interface Forma { double area(); }
sealed interface Animal permits Gato, Cao {}

// Coleções
List<String> l1 = new ArrayList<>();    List<String> l2 = List.of("a", "b");
Set<Integer> s = new HashSet<>();       Map<String, Integer> m = new HashMap<>();
m.put("k", 1);  m.getOrDefault("z", 0);  m.merge("k", 1, Integer::sum);

// Lambdas e Streams
lista.stream().filter(x -> x > 0).map(x -> x * 2).sorted().toList();
lista.stream().collect(Collectors.groupingBy(Pedido::cliente, Collectors.counting()));

// Optional
opt.map(Cliente::nome).orElse("—");     opt.orElseThrow();

// Exceções e recursos
try (var in = Files.newBufferedReader(p)) { } catch (IOException e) { } finally { }
throw new IllegalArgumentException("mensagem");

// Pattern matching
if (o instanceof String s && !s.isEmpty()) { }
return switch (forma) { case Circulo(var r) -> Math.PI * r * r; case Quadrado(var l) -> l * l; };

// Concorrência
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) { ex.submit(() -> tarefa()); }
```

| Comando | Uso |
| --- | --- |
| `javac Arquivo.java` | Compila |
| `java Classe` / `java Arquivo.java` | Executa a classe / compila e executa o arquivo |
| `jshell` | Terminal interativo |
| `jar --create --file app.jar -C classes .` | Empacota |
| `javadoc -d docs src/**/*.java` | Gera documentação |
| `jcmd`, `jps`, `jfr` | Diagnóstico de processos em execução |
| `mvn package` / `./gradlew build` | Build completo do projeto |

### B. Glossário

| Termo | Definição |
| --- | --- |
| Autoboxing | Conversão automática entre primitivo e wrapper (`int` ↔ `Integer`) |
| Bean | Objeto criado e gerenciado pelo contêiner do Spring |
| BOM | Arquivo que fixa as versões de um conjunto de dependências |
| Bytecode | Instruções independentes de plataforma geradas pelo `javac`, executadas pela JVM |
| Checked exception | Exceção que o compilador obriga a tratar ou declarar |
| Classpath | Lista de diretórios e `.jar` onde a JVM procura classes |
| DTO | Objeto que só transporta dados entre camadas, geralmente um record |
| Encapsulamento | Esconder o estado interno e expor apenas operações válidas |
| Garbage collector | Componente da JVM que libera a memória de objetos inalcançáveis |
| Generics | Parâmetros de tipo (`List<T>`) verificados em tempo de compilação |
| Heap | Área de memória compartilhada onde vivem os objetos |
| Imutável | Objeto cujo estado não muda depois de criado |
| Injeção de dependências | Receber as dependências de fora (normalmente no construtor) em vez de criá-las |
| Interface funcional | Interface com um único método abstrato, implementável por lambda |
| JAR | Arquivo compactado com classes e recursos de uma aplicação ou biblioteca |
| JDK / JRE / JVM | Kit de desenvolvimento / ambiente de execução / máquina virtual |
| JEP | JDK Enhancement Proposal: a proposta formal de cada novo recurso do Java |
| JIT | Compilador que transforma bytecode frequente em código nativo durante a execução |
| JPA | Especificação de mapeamento entre objetos e tabelas relacionais |
| LTS | Versão com suporte de longo prazo, lançada a cada dois anos |
| Polimorfismo | Objetos de tipos diferentes respondendo à mesma chamada, cada um à sua maneira |
| Record | Classe imutável de dados com construtor, acessores, `equals`, `hashCode` e `toString` gerados |
| Sealed | Tipo que restringe quais classes podem estendê-lo |
| Stack trace | Lista das chamadas ativas no momento de uma exceção |
| Stream | Pipeline de operações declarativas sobre uma sequência de elementos |
| Virtual thread | Thread leve gerenciada pela JVM, ideal para tarefas que esperam I/O |

### Fontes

- [Oracle Java SE Support Roadmap](https://www.oracle.com/java/technologies/java-se-support-roadmap.html) — versões e calendário LTS
- [JUnit 6 — notas de versão](https://docs.junit.org/6.0.0/release-notes/)
- [Spring Boot 4.1.0](https://spring.io/blog/2026/06/10/spring-boot-4/)
- [dev.java](https://dev.java) e [openjdk.org](https://openjdk.org) — documentação oficial e JEPs
