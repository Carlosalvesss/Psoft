Ordenador



import java.util.ArrayList;
import java.util.List;

/**
 * Papel "Contexto" do padrao Strategy (a classe cliente do enunciado).
 *
 * Mantem uma referencia para a estrategia de ordenacao corrente e permite
 * troca-la livremente em tempo de execucao. Caso a estrategia selecionada
 * recuse o processamento (limite violado), o contexto recorre automaticamente
 * a estrategia padrao (Merge Sort) como fallback e registra a substituicao.
 */
public class Ordenador {

    private EstrategiaOrdenacao estrategia;

    /** Estrategia padrao/fallback, fixa, conforme o enunciado: Merge Sort. */
    private final EstrategiaOrdenacao fallback = new MergeSort();

    public Ordenador(EstrategiaOrdenacao estrategia) {
        this.estrategia = estrategia;
    }

    /** Troca a estrategia de ordenacao em tempo de execucao. */
    public void setEstrategia(EstrategiaOrdenacao estrategia) {
        this.estrategia = estrategia;
    }

    public EstrategiaOrdenacao getEstrategia() {
        return estrategia;
    }

    /**
     * Ordena a lista usando a estrategia corrente. A lista recebida NUNCA e
     * modificada: a operacao trabalha sempre sobre uma copia defensiva. Se a
     * estrategia recusar (LimiteExcedidoException), aciona o fallback.
     *
     * @param elementos lista de inteiros a ordenar (imutavel para este metodo).
     * @return nova lista ordenada.
     */
    public List<Integer> ordena(List<Integer> elementos) {
        // Copia defensiva no limite do contexto: protege a lista original.
        List<Integer> copia = new ArrayList<>(elementos);
        try {
            return estrategia.ordena(copia);
        } catch (LimiteExcedidoException e) {
            System.out.println("[Contexto] " + e.getMessage());
            System.out.println("[Contexto] Estrategia '" + estrategia.getNome()
                    + "' substituida pela estrategia padrao '" + fallback.getNome()
                    + "' (fallback).");
            return fallback.ordena(copia);
        }
    }
}






EstrategiaOrdenacao



import java.util.ArrayList;
import java.util.List;

/**
 * Papel "Estrategia" do padrao Strategy: encapsula um algoritmo de ordenacao
 * intercambiavel, exposto pelo metodo {@link #ordena(List)}.
 *
 * A verificacao do limite e a copia defensiva ficam centralizadas aqui; cada
 * subclasse define apenas o seu limite, o seu nome e o algoritmo em si.
 */
public abstract class EstrategiaOrdenacao {

    /**
     * Operacao do Strategy. Verifica o limite definido pela estrategia e, se
     * estiver dentro do permitido, ordena SEMPRE sobre uma copia defensiva,
     * preservando a lista recebida.
     *
     * @param elementos lista a ordenar (nunca e modificada).
     * @return nova lista ordenada.
     * @throws LimiteExcedidoException se a quantidade exceder o limite.
     */
    public final List<Integer> ordena(List<Integer> elementos) {
        int quantidade = elementos.size();
        if (quantidade > getLimite()) {
            throw new LimiteExcedidoException(getNome(), quantidade, getLimite());
        }
        // Copia defensiva: a ordenacao nunca opera sobre a lista original.
        List<Integer> copia = new ArrayList<>(elementos);
        return ordenar(copia);
    }

    /**
     * Algoritmo de ordenacao propriamente dito, implementado por cada subclasse.
     * Conforme o enunciado, os algoritmos nao precisam ser implementados de fato.
     *
     * @param copia copia defensiva, livre para ser modificada.
     * @return a lista ordenada.
     */
    protected abstract List<Integer> ordenar(List<Integer> copia);

    /**
     * @return quantidade maxima de elementos que a estrategia aceita ordenar.
     *         Acima desse valor, a estrategia recusa o processamento.
     */
    public abstract int getLimite();

    /** @return nome legivel da estrategia (para os logs). */
    public abstract String getNome();
}





BubbleSort


import java.util.List;

/**
 * Estrategia concreta. Algoritmo O(n^2): adequado apenas a listas pequenas,
 * por isso adota um limite baixo de elementos.
 */
public class BubbleSort extends EstrategiaOrdenacao {

    private static final int LIMITE = 1_000;

    @Override
    protected List<Integer> ordenar(List<Integer> copia) {
        // Algoritmo nao implementado (foco no padrao de projeto).
        System.out.println("[Bubble Sort] ordenando " + copia.size() + " elemento(s).");
        return copia;
    }

    @Override
    public int getLimite() {
        return LIMITE;
    }

    @Override
    public String getNome() {
        return "Bubble Sort";
    }
}




InsertionSort


import java.util.List;

/**
 * Estrategia concreta. Algoritmo O(n^2), eficiente em listas pequenas/
 * parcialmente ordenadas; adota um limite intermediario.
 */
public class InsertionSort extends EstrategiaOrdenacao {

    private static final int LIMITE = 5_000;

    @Override
    protected List<Integer> ordenar(List<Integer> copia) {
        System.out.println("[Insertion Sort] ordenando " + copia.size() + " elemento(s).");
        return copia;
    }

    @Override
    public int getLimite() {
        return LIMITE;
    }

    @Override
    public String getNome() {
        return "Insertion Sort";
    }
}




QuickSort


import java.util.List;

/**
 * Estrategia concreta. Algoritmo O(n log n) no caso medio, suporta listas
 * grandes; adota um limite alto.
 */
public class QuickSort extends EstrategiaOrdenacao {

    private static final int LIMITE = 1_000_000;

    @Override
    protected List<Integer> ordenar(List<Integer> copia) {
        System.out.println("[Quick Sort] ordenando " + copia.size() + " elemento(s).");
        return copia;
    }

    @Override
    public int getLimite() {
        return LIMITE;
    }

    @Override
    public String getNome() {
        return "Quick Sort";
    }
}



MergeSort



import java.util.List;

/**
 * Estrategia concreta e estrategia PADRAO (fallback). Por ser o fallback
 * universal, define o maior limite possivel e nunca recusa o processamento.
 */
public class MergeSort extends EstrategiaOrdenacao {

    private static final int LIMITE = Integer.MAX_VALUE;

    @Override
    protected List<Integer> ordenar(List<Integer> copia) {
        System.out.println("[Merge Sort] ordenando " + copia.size() + " elemento(s).");
        return copia;
    }

    @Override
    public int getLimite() {
        return LIMITE;
    }

    @Override
    public String getNome() {
        return "Merge Sort";
    }
}





LimiteExcedidoException


/**
 * Lancada por uma estrategia quando o limite de operacao e violado e ela
 * recusa o processamento da lista. O contexto captura esta excecao para
 * acionar a estrategia de fallback (Merge Sort).
 */
public class LimiteExcedidoException extends RuntimeException {

    public LimiteExcedidoException(String estrategia, int quantidade, int maximo) {
        super("A estrategia '" + estrategia + "' recusou o processamento: "
                + quantidade + " elementos excedem o limite maximo de " + maximo + ".");
    }
}



Bootstrap


import java.util.ArrayList;
import java.util.List;

/**
 * Classe de inicializacao / demonstracao do sistema.
 */
public class Bootstrap {

    public static void main(String[] args) {
        System.out.println("Projeto de Software - Atividade 5 (Strategy)\n");

        // Lista de entrada (imutavel do ponto de vista do cliente).
        List<Integer> dados = new ArrayList<>(List.of(5, 3, 8, 1, 9, 2, 7));

        // Contexto iniciado com Bubble Sort (limite = 1.000).
        Ordenador ordenador = new Ordenador(new BubbleSort());

        System.out.println("== Caso 1: dentro do limite ==");
        List<Integer> r1 = ordenador.ordena(dados);
        System.out.println("Resultado: " + r1);
        System.out.println("Lista original preservada? " + dados.equals(List.of(5, 3, 8, 1, 9, 2, 7)) + "\n");

        // Troca de estrategia em tempo de execucao.
        System.out.println("== Caso 2: troca para Quick Sort ==");
        ordenador.setEstrategia(new QuickSort());
        System.out.println("Resultado: " + ordenador.ordena(dados) + "\n");

        // Caso 3: estrategia recusa -> fallback automatico para Merge Sort.
        System.out.println("== Caso 3: limite violado -> fallback ==");
        ordenador.setEstrategia(new BubbleSort()); // limite 1.000
        List<Integer> grande = gerarLista(1_500);  // 1.500 > 1.000
        List<Integer> r3 = ordenador.ordena(grande);
        System.out.println("Tamanho do resultado: " + r3.size());
    }

    private static List<Integer> gerarLista(int n) {
        List<Integer> lista = new ArrayList<>(n);
        for (int i = n; i > 0; i--) {
            lista.add(i);
        }
        return lista;
    }
}

