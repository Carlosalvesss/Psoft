ProdutoEvent.java



public class ProdutoEvent {
    private Produto source;

    public ProdutoEvent(Produto source) {
        this.source = source;
    }

    public Produto getSource() {
        return source;
    }
}





ProdutoListener.java




public interface ProdutoListener {
    void produtoFicouDisponivel(ProdutoEvent event);
    void produtoEntrouEmPromocao(ProdutoEvent event);
}






ProdutoAdapter.java




public abstract class ProdutoAdapter implements ProdutoListener {
    public void produtoFicouDisponivel(ProdutoEvent event) {
    }
    public void produtoEntrouEmPromocao(ProdutoEvent event) {
    }
}






Produto.java




import java.util.Collection;
import java.util.HashSet;

public class Produto {
    private String nome;
    private String link;
    private String codigoPromocional;
    private Collection<ProdutoListener> listeners = new HashSet<>();

    public Produto(String nome, String link) {
        this.nome = nome;
        this.link = link;
    }

    public String getNome() { return nome; }
    public String getLink() { return link; }
    public String getCodigoPromocional() { return codigoPromocional; }

    public synchronized void addListener(ProdutoListener listener) {
        this.listeners.add(listener);
    }

    public synchronized void removeListener(ProdutoListener listener) {
        this.listeners.remove(listener);
    }

    public void setDisponivel() {
        ProdutoEvent event = new ProdutoEvent(this);
        this.disparaProdutoFicouDisponivel(event);
    }

    public void setEmPromocao(String codigo) {
        this.codigoPromocional = codigo;
        ProdutoEvent event = new ProdutoEvent(this);
        this.disparaProdutoEntrouEmPromocao(event);
    }

    private void disparaProdutoFicouDisponivel(ProdutoEvent event) {
        Collection<ProdutoListener> listenersCopy;
        synchronized (this) {
            listenersCopy = new HashSet<>(listeners);
        }
        for (ProdutoListener listener : listenersCopy) {
            listener.produtoFicouDisponivel(event);
        }
    }

    private void disparaProdutoEntrouEmPromocao(ProdutoEvent event) {
        Collection<ProdutoListener> listenersCopy;
        synchronized (this) {
            listenersCopy = new HashSet<>(listeners);
        }
        for (ProdutoListener listener : listenersCopy) {
            listener.produtoEntrouEmPromocao(event);
        }
    }
}







Usuario.java





public class Usuario implements ProdutoListener {
    private String nome;

    public Usuario(String nome) {
        this.nome = nome;
    }

    @Override
    public void produtoFicouDisponivel(ProdutoEvent event) {
        Produto p = event.getSource();
        System.out.println("Usuário " + nome + " notificado: O produto " + p.getNome() + " está DISPONÍVEL! Link: " + p.getLink());
    }

    @Override
    public void produtoEntrouEmPromocao(ProdutoEvent event) {
        Produto p = event.getSource();
        System.out.println("Usuário " + nome + " notificado: O produto " + p.getNome() + " está em PROMOÇÃO! Link: " + p.getLink() + " | Código: " + p.getCodigoPromocional());
    }
}







AgenteLogistica.java




public class AgenteLogistica extends ProdutoAdapter {
    @Override
    public void produtoFicouDisponivel(ProdutoEvent event) {
        System.out.println("Agente Logística notificado: Calculando melhor opção de entrega para " + event.getSource().getNome());
    }
}





AgenteRecomendacao.java




public class AgenteRecomendacao extends ProdutoAdapter {
    @Override
    public void produtoEntrouEmPromocao(ProdutoEvent event) {
        System.out.println("Agente Recomendação notificado: Atualizando lista de recomendados com " + event.getSource().getNome());
    }
}





Bootstrap.java





public class Bootstrap {

    public static void main(String[] args) {
        System.out.println("Projeto de Software");
        
        Produto placaDeVideo = new Produto("RTX 4090", "https://loja.com/rtx4090");
        
        Usuario marcos = new Usuario("Marcos Maia");
        AgenteLogistica logistica = new AgenteLogistica();
        AgenteRecomendacao recomendacao = new AgenteRecomendacao();
        
        placaDeVideo.addListener(marcos);
        placaDeVideo.addListener(logistica);
        placaDeVideo.addListener(recomendacao);
        
        System.out.println("\n--- Simulando produto ficando disponível ---");
        placaDeVideo.setDisponivel();
        
        System.out.println("\n--- Simulando produto entrando em promoção ---");
        placaDeVideo.setEmPromocao("UFCG2026");
    }
}

