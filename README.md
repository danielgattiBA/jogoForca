# jogoForca
Joga da Forca com arquitetura de software
Okay, vamos criar um exemplo prático de um jogo da forca, aplicando alguns padrões para melhorar sua arquitetura.

**Requisitos do Jogo da Forca:**

1.  Um banco de 15 palavras secretas, cada uma com 5 letras.
2.  O jogador tenta adivinhar a palavra letra por letra.
3.  O jogador tem um número limitado de tentativas (ex: 6 erros).
4.  O jogo exibe a palavra com as letras adivinhadas e underscores para as não adivinhadas.
5.  O jogo informa se o jogador ganhou ou perdeu.

---

### 1. Modelagem do Sistema (Componentes Principais)

Precisaremos de:

*   **`WordRepository` (Repositório de Palavras):** Responsável por fornecer as palavras secretas.
*   **`HangmanGame` (Jogo da Forca - Modelo):** Contém a lógica principal do jogo, o estado atual (palavra secreta, letras tentadas, tentativas restantes, etc.).
*   **`GameController` (Controlador do Jogo):** Orquestra o fluxo do jogo, lida com a entrada do usuário e atualiza o Modelo e a Visão.
*   **`ConsoleView` (Visão do Console):** Responsável por exibir informações ao usuário e obter entradas.

---

### 2. Escolha de Padrões

*   **Padrão Arquitetural: MVC (Model-View-Controller) simplificado**
    *   **Model:** `HangmanGame` e `WordRepository`. Contém os dados e a lógica de negócios. Não conhece a UI.
    *   **View:** `ConsoleView`. Apresenta os dados do modelo ao usuário e envia os comandos do usuário (como uma letra digitada) para o Controller.
    *   **Controller:** `GameController`. Atua como intermediário entre o Model e a View. Processa a entrada da View, interage com o Model e seleciona a View apropriada para exibir a resposta.

*   **Padrão de Projeto: Repository (para `WordRepository`)**
    *   **Objetivo:** Abstrair a fonte de dados das palavras. Inicialmente, pode ser uma lista em memória, mas poderia ser facilmente alterado para ler de um arquivo ou banco de dados sem impactar o resto do sistema.
    *   **Benefício:** Desacoplamento, testabilidade.

*   **Padrão de Projeto: Observer (entre `HangmanGame` e `ConsoleView`)**
    *   **Objetivo:** Permitir que a `ConsoleView` seja notificada automaticamente quando o estado do `HangmanGame` mudar (ex: após uma tentativa).
    *   **Subject (Observado):** `HangmanGame`.
    *   **Observer (Observador):** `ConsoleView`.
    *   **Benefício:** Desacoplamento. O `HangmanGame` não precisa conhecer os detalhes da `ConsoleView`, apenas notifica seus observadores sobre mudanças. Isso facilita a adição de outras Views no futuro (ex: uma GUI).

---

### 3. Linha de Desenvolvimento e Estrutura do Código (Conceitual em Java/Pseudocódigo)

#### Passo 1: `WordRepository`

```java
// WordRepository.java
import java.util.Arrays;
import java.util.List;
import java.util.Random;

public class WordRepository {
    private final List<String> words = Arrays.asList(
        "CASAS", "LIVRO", "MOUSE", "FLORA", "PRAIA",
        "CORPO", "TEMPO", "SONHO", "VERDE", "AZUIS",
        "NOITE", "FORTE", "SAGAZ", "IDEIA", "JUSTO"
    );
    private final Random random = new Random();

    public String getRandomWord() {
        if (words.isEmpty()) {
            throw new IllegalStateException("Banco de palavras está vazio.");
        }
        return words.get(random.nextInt(words.size()));
    }
}
```

#### Passo 2: `HangmanGame` (Model) - Parte 1 (Estado e Lógica Básica)

```java
// HangmanGame.java
import java.util.HashSet;
import java.util.Set;
import java.util.ArrayList;
import java.util.List;

// Interface para o Observer
interface GameObserver {
    void update(HangmanGame game);
}

public class HangmanGame {
    private String secretWord;
    private Set<Character> guessedLetters;
    private int remainingAttempts;
    private final int maxAttempts = 6;
    private GameStatus status;

    // Para o padrão Observer
    private List<GameObserver> observers = new ArrayList<>();

    public enum GameStatus { PLAYING, WON, LOST }

    public HangmanGame(String secretWord) {
        this.secretWord = secretWord.toUpperCase();
        this.guessedLetters = new HashSet<>();
        this.remainingAttempts = maxAttempts;
        this.status = GameStatus.PLAYING;
    }

    // Métodos para o padrão Observer
    public void addObserver(GameObserver observer) {
        observers.add(observer);
    }

    public void removeObserver(GameObserver observer) {
        observers.remove(observer);
    }

    private void notifyObservers() {
        for (GameObserver observer : observers) {
            observer.update(this);
        }
    }

    public String getMaskedWord() {
        StringBuilder masked = new StringBuilder();
        for (char c : secretWord.toCharArray()) {
            if (guessedLetters.contains(c) || c == ' ') { // Espaços são revelados
                masked.append(c);
            } else {
                masked.append("_");
            }
        }
        return masked.toString();
    }

    public boolean guessLetter(char letter) {
        if (status != GameStatus.PLAYING) {
            return false; // Não processa se o jogo acabou
        }

        letter = Character.toUpperCase(letter);
        if (!Character.isLetter(letter)) {
            // Poderia lançar exceção ou apenas ignorar e não contar como tentativa
            System.out.println("Entrada inválida. Por favor, digite uma letra.");
            notifyObservers(); // Notifica para re-renderizar, se necessário
            return false;
        }

        if (guessedLetters.contains(letter)) {
            // Letra já tentada, não penaliza, mas informa
            System.out.println("Letra '" + letter + "' já tentada.");
            notifyObservers();
            return false;
        }

        guessedLetters.add(letter);
        boolean correctGuess = secretWord.indexOf(letter) >= 0;

        if (!correctGuess) {
            remainingAttempts--;
        }

        updateStatus();
        notifyObservers(); // Notifica observadores sobre a mudança de estado
        return correctGuess;
    }

    private void updateStatus() {
        if (getMaskedWord().equals(secretWord)) {
            status = GameStatus.WON;
        } else if (remainingAttempts <= 0) {
            status = GameStatus.LOST;
        } else {
            status = GameStatus.PLAYING;
        }
    }

    // Getters para a View e Controller
    public int getRemainingAttempts() { return remainingAttempts; }
    public GameStatus getStatus() { return status; }
    public String getSecretWord() { return secretWord; } // Usado para mostrar a palavra no final
    public Set<Character> getGuessedLetters() { return new HashSet<>(guessedLetters); } // Retorna cópia
}
```

#### Passo 3: `ConsoleView` (View)

```java
// ConsoleView.java
import java.util.Scanner;
import java.util.stream.Collectors;

public class ConsoleView implements GameObserver {
    private Scanner scanner = new Scanner(System.in);

    public void displayWelcome() {
        System.out.println("--- Bem-vindo ao Jogo da Forca! ---");
    }

    public void displayGameStatus(HangmanGame game) {
        System.out.println("\nPalavra: " + game.getMaskedWord());
        System.out.println("Tentativas restantes: " + game.getRemainingAttempts());
        String guessed = game.getGuessedLetters().stream()
                             .map(String::valueOf)
                             .collect(Collectors.joining(", "));
        System.out.println("Letras tentadas: [" + guessed + "]");
    }

    public char promptForGuess() {
        System.out.print("Digite uma letra: ");
        String input = scanner.nextLine();
        if (input.length() == 1) {
            return input.charAt(0);
        }
        return ' '; // Indica entrada inválida para o controller lidar
    }

    public void displayWinMessage() {
        System.out.println("\nParabéns! Você venceu!");
    }

    public void displayLossMessage(String secretWord) {
        System.out.println("\nVocê perdeu! A palavra era: " + secretWord);
    }

    public void displayInvalidInput() {
        System.out.println("Entrada inválida. Por favor, digite apenas uma letra.");
    }

    public void displayAlreadyGuessed() {
        System.out.println("Você já tentou essa letra.");
    }
    
    @Override
    public void update(HangmanGame game) {
        // Quando o modelo notifica, a view se atualiza
        clearConsole(); // Simples função para limpar o console (pode variar por OS)
        displayGameStatus(game);

        if (game.getStatus() == HangmanGame.GameStatus.WON) {
            displayWinMessage();
        } else if (game.getStatus() == HangmanGame.GameStatus.LOST) {
            displayLossMessage(game.getSecretWord());
        }
    }

    // Método auxiliar simples para "limpar" o console
    private void clearConsole() {
        // Para um console real, pode ser System.out.print("\033[H\033[2J"); System.out.flush();
        // ou apenas imprimir várias linhas em branco para simular.
        for (int i = 0; i < 10; ++i) System.out.println();
    }
}
```

#### Passo 4: `GameController` (Controller) e `Main`

```java
// GameController.java
public class GameController {
    private HangmanGame gameModel;
    private ConsoleView gameView;
    private WordRepository wordRepository;

    public GameController() {
        this.wordRepository = new WordRepository();
        this.gameView = new ConsoleView();
    }

    public void startGame() {
        String secretWord = wordRepository.getRandomWord();
        this.gameModel = new HangmanGame(secretWord);
        this.gameModel.addObserver(gameView); // View se registra como observadora do Model

        gameView.displayWelcome();
        gameModel.notifyObservers(); // Força a primeira renderização do estado inicial

        play();
    }

    private void play() {
        while (gameModel.getStatus() == HangmanGame.GameStatus.PLAYING) {
            char guess = gameView.promptForGuess();

            if (guess == ' ') { // Entrada inválida da View
                gameView.displayInvalidInput();
                continue;
            }
            
            // Model lida com a lógica de "já tentada" e notifica a view se for o caso
            // A mensagem de "já tentada" será impressa pelo HangmanGame e a view re-renderizada pelo observer
            gameModel.guessLetter(guess);
            // A View é atualizada automaticamente pelo padrão Observer após o guessLetter
        }

        // Mensagem final de vitória/derrota já é tratada pelo update da View
        System.out.println("Obrigado por jogar!");
    }

    public static void main(String[] args) {
        GameController controller = new GameController();
        controller.startGame();
    }
}
```

---

### Benefícios dos Padrões Utilizados:

1.  **MVC:**
    *   **Separação de Responsabilidades:** A lógica do jogo (`HangmanGame`) está separada da apresentação (`ConsoleView`) e do fluxo de controle (`GameController`).
    *   **Testabilidade:** `HangmanGame` pode ser testado independentemente da UI.
    *   **Flexibilidade:** Seria mais fácil trocar a `ConsoleView` por uma `GraphicalView` sem alterar significativamente o `HangmanGame` ou o `GameController`.

2.  **Repository (`WordRepository`):**
    *   **Abstração da Fonte de Dados:** Se decidirmos carregar palavras de um arquivo `palavras.txt` ou de um banco de dados, apenas a implementação de `WordRepository` precisa mudar. O `GameController` e o `HangmanGame` não seriam afetados.

3.  **Observer:**
    *   **Desacoplamento entre Model e View:** `HangmanGame` (Model) não conhece os detalhes de `ConsoleView`. Ele apenas notifica que seu estado mudou.
    *   **Múltiplas Views:** Poderíamos ter várias views (ex: uma `LogView` que registra cada jogada) observando o mesmo `HangmanGame` e reagindo às mudanças de estado de forma independente.
    *   **Reatividade:** A UI reage automaticamente às mudanças no modelo, simplificando o `GameController`, que não precisa se preocupar em explicitamente mandar a View se redesenhar após cada ação.

Este exemplo demonstra como os padrões podem estruturar o código, tornando-o mais organizado, flexível, testável e fácil de manter, mesmo para um jogo relativamente simples como o da Forca.
