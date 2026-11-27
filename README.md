# TRABAJO-FINAL
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.LinkedList;
import java.util.List;
import java.util.Map;
import java.util.PriorityQueue;
import java.util.Queue;

/** Programa de demostración: construye un grafo y ejecuta BFS, DFS, Dijkstra y Floyd-Warshall. */
public class Main {

    public static void main(String[] args) {
        Grafo g = new Grafo(false);
        g.agregarArista("A", "B", 4);
        g.agregarArista("A", "C", 2);
        g.agregarArista("B", "D", 5);
        g.agregarArista("B", "C", 1);
        g.agregarArista("C", "D", 8);
        g.agregarArista("C", "E", 10);
        g.agregarArista("D", "E", 2);
        g.agregarArista("D", "F", 6);
        g.agregarArista("E", "F", 3);

        System.out.println("=== Lista de adyacencia ===");
        g.imprimir();

        System.out.println("\n=== BFS desde A ===");
        ResultadoRecorrido bfs = Recorridos.bfs(g, "A");
        System.out.println("Orden: " + bfs.orden);
        for (int i = 0; i < g.numVertices(); i++)
            System.out.println("  " + g.nombreDe(i) + ": nivel " + bfs.nivel[i]
                    + ", padre " + (bfs.padre[i] < 0 ? "-" : g.nombreDe(bfs.padre[i])));

        System.out.println("\n=== DFS desde A ===");
        ResultadoRecorrido dfs = Recorridos.dfs(g, "A");
        System.out.println("Orden: " + dfs.orden);
        for (int i = 0; i < g.numVertices(); i++)
            System.out.println("  " + g.nombreDe(i) + ": descubierto " + dfs.descubrimiento[i]
                    + ", finalizado " + dfs.finalizacion[i]);

        System.out.println("\n=== Dijkstra desde A ===");
        ResultadoDijkstra dj = CaminosMinimos.dijkstra(g, "A");
        for (int i = 0; i < g.numVertices(); i++)
            System.out.println("  A -> " + g.nombreDe(i) + ": costo " + Grafo.formato(dj.distancia[i])
                    + ", ruta " + nombres(g, dj.camino(i)));

        System.out.println("\n=== Floyd-Warshall (todos los pares) ===");
        ResultadoFloyd fw = CaminosMinimos.floydWarshall(g);
        imprimirMatriz(g, fw.distancia);
        System.out.println("Ruta B -> F: " + nombres(g, fw.camino(g.indiceDe("B"), g.indiceDe("F")))
                + " (costo " + Grafo.formato(fw.distancia[g.indiceDe("B")][g.indiceDe("F")]) + ")");
        System.out.println("¿Ciclo negativo? " + fw.cicloNegativo);

        // Segundo ejemplo: grafo dirigido con un vértice inalcanzable
        System.out.println("\n=== Grafo dirigido (con vértice inalcanzable) ===");
        Grafo h = new Grafo(true);
        h.agregarArista("1", "2", 3);
        h.agregarArista("2", "3", 4);
        h.agregarArista("1", "3", 10);
        h.agregarVertice("4");
        ResultadoDijkstra d2 = CaminosMinimos.dijkstra(h, "1");
        for (int i = 0; i < h.numVertices(); i++) {
            String costo = Double.isInfinite(d2.distancia[i]) ? "inalcanzable" : Grafo.formato(d2.distancia[i]);
            System.out.println("  1 -> " + h.nombreDe(i) + ": " + costo);
        }
    }

    private static String nombres(Grafo g, List<Integer> ruta) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < ruta.size(); i++) {
            if (i > 0) sb.append(" -> ");
            sb.append(g.nombreDe(ruta.get(i)));
        }
        return sb.toString();
    }

    private static void imprimirMatriz(Grafo g, double[][] d) {
        int n = g.numVertices();
        StringBuilder cab = new StringBuilder("     ");
        for (int j = 0; j < n; j++) cab.append(String.format("%5s", g.nombreDe(j)));
        System.out.println(cab);
        for (int i = 0; i < n; i++) {
            StringBuilder fila = new StringBuilder(String.format("%5s", g.nombreDe(i)));
            for (int j = 0; j < n; j++)
                fila.append(String.format("%5s", Double.isInfinite(d[i][j]) ? "INF" : Grafo.formato(d[i][j])));
            System.out.println(fila);
        }
    }
}

/** Arista dirigida y ponderada: guarda el índice del vértice destino y su peso. */
class Arista {
    private final int destino;
    private final double peso;

    public Arista(int destino, double peso) {
        this.destino = destino;
        this.peso = peso;
    }

    public int getDestino() { return destino; }
    public double getPeso() { return peso; }
}

/** Vértice del grafo: tiene un índice, un nombre y su lista de aristas salientes. */
class Vertice {
    private final int indice;
    private final String nombre;
    private final List<Arista> aristas = new ArrayList<>();

    public Vertice(int indice, String nombre) {
        this.indice = indice;
        this.nombre = nombre;
    }

    public void agregarArista(Arista a) { aristas.add(a); }

    public int getIndice() { return indice; }
    public String getNombre() { return nombre; }
    public List<Arista> getAristas() { return aristas; }
}

/**
 * Grafo ponderado representado con listas de adyacencia.
 * Puede ser dirigido o no dirigido. Es la clase central: los algoritmos
 * (Recorridos y CaminosMinimos) solo leen su estructura.
 */
class Grafo {
    private final boolean dirigido;
    private final List<Vertice> vertices = new ArrayList<>();
    private final Map<String, Integer> indicePorNombre = new HashMap<>();

    public Grafo(boolean dirigido) {
        this.dirigido = dirigido;
    }

    /** Agrega un vértice si no existe y devuelve su índice. */
    public int agregarVertice(String nombre) {
        Integer existente = indicePorNombre.get(nombre);
        if (existente != null) return existente;
        int idx = vertices.size();
        vertices.add(new Vertice(idx, nombre));
        indicePorNombre.put(nombre, idx);
        return idx;
    }

    /** Agrega una arista (y su inversa si el grafo no es dirigido). Crea los vértices si faltan. */
    public void agregarArista(String origen, String destino, double peso) {
        int o = agregarVertice(origen);
        int d = agregarVertice(destino);
        vertices.get(o).agregarArista(new Arista(d, peso));
        if (!dirigido) {
            vertices.get(d).agregarArista(new Arista(o, peso));
        }
    }

    public int indiceDe(String nombre) {
        Integer idx = indicePorNombre.get(nombre);
        if (idx == null) throw new IllegalArgumentException("Vértice inexistente: " + nombre);
        return idx;
    }

    public String nombreDe(int indice) { return vertices.get(indice).getNombre(); }
    public Vertice getVertice(int indice) { return vertices.get(indice); }
    public int numVertices() { return vertices.size(); }
    public boolean esDirigido() { return dirigido; }

    /** Imprime la lista de adyacencia. */
    public void imprimir() {
        for (Vertice v : vertices) {
            StringBuilder sb = new StringBuilder(v.getNombre() + " -> ");
            for (Arista a : v.getAristas()) {
                sb.append(nombreDe(a.getDestino())).append("(").append(formato(a.getPeso())).append(") ");
            }
            System.out.println(sb.toString().trim());
        }
    }

    static String formato(double x) {
        return x == (long) x ? String.valueOf((long) x) : String.valueOf(x);
    }
}

/** Resultado de BFS o DFS: orden de visita, padre de cada vértice y nivel (BFS) o tiempos (DFS). */
class ResultadoRecorrido {
    public final List<String> orden = new ArrayList<>();
    public final int[] padre;
    public final int[] nivel;          // BFS: distancia en aristas desde el origen
    public final int[] descubrimiento; // DFS: tiempo en que se descubre el vértice
    public final int[] finalizacion;   // DFS: tiempo en que se termina de explorar

    public ResultadoRecorrido(int n) {
        padre = new int[n];
        nivel = new int[n];
        descubrimiento = new int[n];
        finalizacion = new int[n];
        Arrays.fill(padre, -1);
        Arrays.fill(nivel, -1);
    }
}

/** Algoritmos de recorrido: búsqueda en anchura (BFS) y en profundidad (DFS). */
class Recorridos {

    /** BFS desde un vértice origen: visita por niveles usando una cola (FIFO). */
    public static ResultadoRecorrido bfs(Grafo g, String origen) {
        int n = g.numVertices();
        ResultadoRecorrido r = new ResultadoRecorrido(n);
        boolean[] visitado = new boolean[n];
        Queue<Integer> cola = new ArrayDeque<>();

        int s = g.indiceDe(origen);
        visitado[s] = true;
        r.nivel[s] = 0;
        cola.add(s);

        while (!cola.isEmpty()) {
            int u = cola.poll();
            r.orden.add(g.nombreDe(u));
            for (Arista a : g.getVertice(u).getAristas()) {
                int v = a.getDestino();
                if (!visitado[v]) {
                    visitado[v] = true;
                    r.padre[v] = u;
                    r.nivel[v] = r.nivel[u] + 1;
                    cola.add(v);
                }
            }
        }
        return r;
    }

    /** DFS desde un vértice origen: baja lo más posible por cada rama (recursión = pila). */
    public static ResultadoRecorrido dfs(Grafo g, String origen) {
        int n = g.numVertices();
        ResultadoRecorrido r = new ResultadoRecorrido(n);
        boolean[] visitado = new boolean[n];
        int[] reloj = {0};
        dfsVisitar(g, g.indiceDe(origen), visitado, r, reloj);
        return r;
    }

    private static void dfsVisitar(Grafo g, int u, boolean[] visitado, ResultadoRecorrido r, int[] reloj) {
        visitado[u] = true;
        r.descubrimiento[u] = ++reloj[0];
        r.orden.add(g.nombreDe(u));
        for (Arista a : g.getVertice(u).getAristas()) {
            int v = a.getDestino();
            if (!visitado[v]) {
                r.padre[v] = u;
                dfsVisitar(g, v, visitado, r, reloj);
            }
        }
        r.finalizacion[u] = ++reloj[0];
    }
}

/** Resultado de Dijkstra: distancia mínima y predecesor de cada vértice respecto al origen. */
class ResultadoDijkstra {
    public final int origen;
    public final double[] distancia;
    public final int[] predecesor;

    public ResultadoDijkstra(int origen, int n) {
        this.origen = origen;
        this.distancia = new double[n];
        this.predecesor = new int[n];
        Arrays.fill(distancia, Double.POSITIVE_INFINITY);
        Arrays.fill(predecesor, -1);
        distancia[origen] = 0;
    }

    /** Reconstruye el camino origen -> destino siguiendo los predecesores hacia atrás. */
    public List<Integer> camino(int destino) {
        if (Double.isInfinite(distancia[destino])) return Collections.emptyList();
        LinkedList<Integer> ruta = new LinkedList<>();
        for (int v = destino; v != -1; v = predecesor[v]) ruta.addFirst(v);
        return ruta;
    }
}

/** Resultado de Floyd-Warshall: matriz de distancias y matriz "siguiente" para reconstruir rutas. */
class ResultadoFloyd {
    public final double[][] distancia;
    public final int[][] siguiente;
    public final boolean cicloNegativo;

    public ResultadoFloyd(double[][] distancia, int[][] siguiente, boolean cicloNegativo) {
        this.distancia = distancia;
        this.siguiente = siguiente;
        this.cicloNegativo = cicloNegativo;
    }

    /** Camino más corto entre dos vértices (lista vacía si no hay ruta). */
    public List<Integer> camino(int i, int j) {
        if (siguiente[i][j] == -1) return Collections.emptyList();
        List<Integer> ruta = new ArrayList<>();
        ruta.add(i);
        while (i != j) {
            i = siguiente[i][j];
            ruta.add(i);
        }
        return ruta;
    }
}

/** Algoritmos de costo mínimo: Dijkstra (un origen) y Floyd-Warshall (todos los pares). */
class CaminosMinimos {

    /** Entrada de la cola de prioridad: vértice con la distancia con la que fue encolado. */
    private static class Entrada implements Comparable<Entrada> {
        final int vertice;
        final double distancia;
        Entrada(int vertice, double distancia) { this.vertice = vertice; this.distancia = distancia; }
        @Override public int compareTo(Entrada o) { return Double.compare(distancia, o.distancia); }
    }

    /** Dijkstra con cola de prioridad. Requiere pesos no negativos. */
    public static ResultadoDijkstra dijkstra(Grafo g, String origen) {
        int n = g.numVertices();
        int s = g.indiceDe(origen);
        ResultadoDijkstra r = new ResultadoDijkstra(s, n);
        boolean[] definitivo = new boolean[n];
        PriorityQueue<Entrada> cola = new PriorityQueue<>();
        cola.add(new Entrada(s, 0));

        while (!cola.isEmpty()) {
            Entrada e = cola.poll();
            int u = e.vertice;
            if (definitivo[u]) continue;      // entrada obsoleta
            definitivo[u] = true;

            for (Arista a : g.getVertice(u).getAristas()) {
                if (a.getPeso() < 0)
                    throw new IllegalArgumentException("Dijkstra no admite pesos negativos");
                int v = a.getDestino();
                double nueva = r.distancia[u] + a.getPeso();
                if (nueva < r.distancia[v]) {  // relajación
                    r.distancia[v] = nueva;
                    r.predecesor[v] = u;
                    cola.add(new Entrada(v, nueva));
                }
            }
        }
        return r;
    }

    /** Floyd-Warshall: distancias mínimas entre todos los pares de vértices. O(V^3). */
    public static ResultadoFloyd floydWarshall(Grafo g) {
        int n = g.numVertices();
        double[][] d = new double[n][n];
        int[][] sig = new int[n][n];
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                d[i][j] = (i == j) ? 0 : Double.POSITIVE_INFINITY;
                sig[i][j] = (i == j) ? i : -1;
            }
        }
        for (int u = 0; u < n; u++) {
            for (Arista a : g.getVertice(u).getAristas()) {
                int v = a.getDestino();
                if (a.getPeso() < d[u][v]) {
                    d[u][v] = a.getPeso();
                    sig[u][v] = v;
                }
            }
        }
        for (int k = 0; k < n; k++) {
            for (int i = 0; i < n; i++) {
                if (Double.isInfinite(d[i][k])) continue;
                for (int j = 0; j < n; j++) {
                    if (d[i][k] + d[k][j] < d[i][j]) {
                        d[i][j] = d[i][k] + d[k][j];
                        sig[i][j] = sig[i][k];
                    }
                }
            }
        }
        boolean negativo = false;
        for (int i = 0; i < n; i++) if (d[i][i] < 0) negativo = true;
        return new ResultadoFloyd(d, sig, negativo);
    }
}
