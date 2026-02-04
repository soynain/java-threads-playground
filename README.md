# java-threads-playground

En este repo jugaré con todas las clases de hilos que hayan en java, así como las estructuras
de datos thread safe para saber yo como puedo aplicarlos.


Lo primero será meterme a lo fundamental de los hilos.

La última vez que usé hilos fue en un tamagochi que me costó un mes y medio programar, con componentes dinámicos en SWING
y todo el pex, con una laptop de 4 gigas de RAM. Usaba los hilos para instanciar un objeto con su propio cronometro y descuento o algo así,
y cada tamagochi tenía una velocidad y una barra de vida que disminuia de manera más rápida o lenta.

En hilos, el más clásico es por Runnable, por interfaz es más fácil.

Hice esta prueba, con peticiones bloqueantes:


````app.java
public static void main(String[] args) throws IOException, InterruptedException {
        Thread thread = new Thread(new App());
        Thread thread2 = new Thread(new App());
        thread.start();
        thread2.start();
        thread.join();
        thread2.join();
    }

    @Override
    public void run() {
        // TODO Auto-generated method stub
        for (int i = 0; i < 5; i++) {
           // System.out.println("Hello World! ");
            HttpRequest request = HttpRequest.newBuilder(URI.create("https://google.com")).GET().build();

            HttpResponse<String> client;
            try {
                client = HttpClient.newHttpClient().send(request, HttpResponse.BodyHandlers.ofString());
                this.globalCounter++;
                System.out.println(client.statusCode()+">"+i+" >>"+Thread.currentThread().getName()+" global"+this.globalCounter);
            } catch (IOException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            
        }
    }
````

Y su output:
````app.sh
301>0 >>Thread-0 global2
301>0 >>Thread-1 global1
301>1 >>Thread-1 global3
301>1 >>Thread-0 global4
301>2 >>Thread-1 global5
301>2 >>Thread-0 global6
301>3 >>Thread-0 global7
301>3 >>Thread-1 global8
301>4 >>Thread-0 global9
301>4 >>Thread-1 global10
````

volatile son variables de lectura para que los hilos sincronizadamente, actualizen un valor en común.
Dentro del scope del run cada hilo tiene su propio estado y variables locales dentro de un ciclo run. En este
caso hacemos peticiones sincronas, 4 veces, el volatile guarda el counter global por hilo pero dentro de run el contador
i es independiente.

Le pedí a chatgpt que me prepara un problemita para aplicar la clase thread normal y me estoy adentrando a un mundo interesante.

Puntos interesantes:

````main.java
System.out.println(Thread.activeCount()+" numeros de hilos antes de iniciar");
System.out.println(ManagementFactory.getThreadMXBean().getThreadCount()+" otra medida mas preciza");
System.out.println("UNIVERSO DE LA MAQUINA: "+ Runtime.getRuntime().availableProcessors());
````

El primero toma el número de hilos activos, solo del thread group main, ignora system u otros, el segundo
ignora ese limite y si te lista todos los hilos daemon y no daemon

Cada hilo tiene un método .setDaemon(), en resumidas, si lo pones en true y tu método main se acaba, se morirá
con ese hilo, de lo contrario tu programa puede quedarse colgado.

El tercero por máximo te dará 16 procesadores disponibles.

En un escenario de hilos depende de tu diseño: si harás peticiones con calculo donde se esperen respuestas o hagas llamados de datos paralelos, puedes
asignar muchos hilos, de lo contrario quedate solo con el límite de los 16.

Al crear un hilo, ocupa un mb en la RAM, *existen otras clases para mejorar ese proceso de recursos*.

````OrderThreads.java
public class OrderThreads implements Runnable{

    private static volatile Integer orderStock = 100;

    private static ReentrantLock lock = new ReentrantLock();

    /**Medida general del tiempo, si lo ejecutas en main, no se puede */

    private volatile long startTime = System.currentTimeMillis();

    private volatile long endTime = 0L;

    private volatile long totalTime = 0L;

    private long TO_SECONDS = 1000;
    
    @Override
    public void run(){
        while (true) {
            /**En base a un recurso, se sincroniza, por estándar aunque sea volatile
             * o atomic, lo wrapeas con ese bloque para evitar inconcistencias
             */
           // synchronized (orderStock) {
            lock.lock();
            System.out.println("numero de hilos actuales: "+Thread.activeCount());
            try {
                if (orderStock > 0) {
                    Order order = new Order(UUID.randomUUID().toString());
                    System.out.println("Processing order ID: " + order.getId() + " with processing time: " + order.getProcessingTime() + "ms with thread: " + Thread.currentThread().getName());
                    try {
                        Thread.sleep(order.getProcessingTime());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    orderStock--;
                    System.out.println("Completed order ID: " + order.getId() + ". Remaining stock: " + orderStock+". Processed by thread: " + Thread.currentThread().getName());
                }else{
                    System.out.println("No more orders to process.");
                    endTime = System.currentTimeMillis();
                    totalTime = endTime - startTime;

                    System.out.println("Total execution time: " + totalTime / TO_SECONDS + "ms");
                    break;
                }
            } finally{
                lock.unlock();
            }
          //  }
        }
    }
    
}
````
En este ejemplo, simulamos un procesamiento de ordenes, con un sleep generado aleatoriamente. Con 51 hilos para 100 ordenes toma 47 segundos. Si lo hicieras líneal
obvio tomaría más tiempo. Esa es la simple útilidad de los hilos. Son para ETL'S de vieja escuela o procesos donde requieras enviar multiples peticiones.

El lock funciona mejor que el syncronized para el bloqueo de recursos volatiles y atómicos.

<img width="578" height="492" alt="image" src="https://github.com/user-attachments/assets/84b819a0-a64b-4136-be17-dd639289a96c" />

<img width="1027" height="482" alt="image" src="https://github.com/user-attachments/assets/d710ffd6-44ea-4b9f-bff5-66b639370560" />


Ahora, toda esta medología aborda el control de los hilos a mano, un tema que se manejó desde java 5, el mejor handler para esos
escenarios es el ExecutorService, con diferentes estrategias. Pero no nos adelantemos.

Antes de concluír el escenario, la estrategia del Runnable y Thread no es mala, pero solo aplica en escenarios
donde si sabes cuantos hilos vas a ocupar.

Mencioné hace tiempo que hice un Tamagochi que pueden encontrar aqui:

https://github.com/soynain/tamagochiJavaTap/blob/master/src/newpackage/Mascota.java

Vean el archivo y notarán que ocupo muchas banderas, para esto:

<img width="1979" height="1094" alt="image" src="https://github.com/user-attachments/assets/9313576d-708b-4de7-91ec-c3ee05d6635b" />

Esto es código puerco, lo que hacia en esta clase era generar componentes dinámicos en Swing (aquí avalo que era un dios con java desde la uni jejejeje)

y en base a una bandera global, si las estadisticas del tamagochi llegaban a 0, ya moría el hilo y ese tamagochi se moría.

<img width="1958" height="1002" alt="image" src="https://github.com/user-attachments/assets/b0ed7987-a414-4f35-8369-7db7b4921f59" />

Por cada iteración del hilo dormías el hilo 5 segundos y restabas al azar en el tamagochi sus estadisticas, y como
el hilo encapsulaba sus propias variables, con métodos set vinculados al componente SWING, aumentabas
las estadisticas del hilo:

<img width="1097" height="548" alt="image" src="https://github.com/user-attachments/assets/6cedcd13-6422-4772-a499-7f0b488adff7" />

En epocas donde no existía chat gpt, hacer esto te hacia sentir un dios, y ciertamente era problemático ver o saber
como hacer componentes dinámicos en algo tan estrecho como SWING, combinando un layout hecho con el interface constructor y
combinando el layout declarativo.

Y se añadia la grilla con el declarativo de swing e iniciabas tu instancia:

<img width="1422" height="765" alt="image" src="https://github.com/user-attachments/assets/fbaf1565-39e8-4ce0-8ce1-6ee3fdede1d5" />

Las acciones del tamagochi con el hilo eran declarativas, y con un action listener tus métodos vivian internamente:

<img width="1624" height="891" alt="image" src="https://github.com/user-attachments/assets/5f3921b0-a480-47d5-b421-53e6b627b85a" />

Un código espantoso pero funcional.

Nada del otro mundo estos métodos, también habia hecho una galeria de imagenes en java, que transicionaba sus hilos de
manera rápida, lo pueden checar acá:

<img width="1624" height="891" alt="image" src="https://github.com/user-attachments/assets/2772a9be-a301-4215-9cad-9d7a5ce93a6e" />

El chiste recaía en que si presionabas un botón de 1x a 4x, aumentaba la velocidad.

El método sucio era declarar 4 hilos, solo despertar uno, si seleccionaban x velocidad, detenías todos e iniciabas el hilo
con un Thread.sleep menor

<img width="2559" height="1439" alt="image" src="https://github.com/user-attachments/assets/16527a9c-5021-4ed9-a5be-6e4276e6be13" />

Serie 4 tiene el menor thread.sleep

````main.java
class serieCuatro implements Runnable {

    public boolean iterar;
    public JLabel galy;
    fileInstancia global;
    public JRadioButton unoX, dosX, tresX, cuatroX;
    public JButton leer, iniciar, bocina;
    ArrayList<ImageIcon> labelCambio;
    serieCuatro(JLabel lblGaleria, JRadioButton uno, JRadioButton dos, JRadioButton tres, JRadioButton cuatro, JButton ini, JButton boci) {
        this.galy = lblGaleria;
        global = new fileInstancia();
        this.unoX = uno;
        this.dosX = dos;
        this.tresX = tres;
        this.cuatroX = cuatro;
        this.bocina = boci;
        this.iniciar = ini;
        labelCambio=new ArrayList();
    }

    @Override
    public void run() {
        while (iterar != false && global.getLongitudArr() >= global.getContadorImg()) {
            galy.setIcon(new ImageIcon(new ImageIcon(global.indexArreglo(global.getContadorImg()).getAbsolutePath()).getImage().getScaledInstance(galy.getWidth(), galy.getHeight(), Image.SCALE_SMOOTH)));
            global.aumentarCont();
            if (global.getContadorImg() > global.getLongitudArr()) {
                detener();
                JOptionPane.showMessageDialog(galy, "CREDITOS: \n miembros del equipo: el de los chescos: Soto Guzmán Moisés Nain \n la que editó las fotos: Maria Fernanda Rodriguez Rivas \n el creativo detrás de la historia: Ruben Aguilar Nuñez Alexis");
                global.resetInstancia();
                resetGaleryGui();
            try {
                Thread.sleep(30);
            } catch (InterruptedException ex) {
                //Thread.currentThread().interrupt();
                System.out.println(ex);
            }
        }
    }
    }
````


Serie dos tiene el mayor:
````main.java

class serieDos implements Runnable {

    public boolean iterar;
    public JLabel galy;
    fileInstancia global;
    public JRadioButton unoX, dosX, tresX, cuatroX;
    public JButton leer, iniciar, bocina;

    serieDos(JLabel lblGaleria, JRadioButton uno, JRadioButton dos, JRadioButton tres, JRadioButton cuatro, JButton ini, JButton boci) {
        this.galy = lblGaleria;
        global = new fileInstancia();
        this.unoX = uno;
        this.dosX = dos;
        this.tresX = tres;
        this.cuatroX = cuatro;
        this.bocina = boci;
        this.iniciar = ini;
    }

    @Override
    public void run() {
        while (iterar != false && global.getLongitudArr() >= global.getContadorImg()) {
            galy.setIcon(new ImageIcon(new ImageIcon(global.indexArreglo(global.getContadorImg()).getAbsolutePath()).getImage().getScaledInstance(galy.getWidth(), galy.getHeight(), Image.SCALE_SMOOTH)));
            global.aumentarCont();
            if (global.getContadorImg() > global.getLongitudArr()) {
                //galy.setText(hour + ":" + (minute) + ":" + second + "->" + global.getContadorImg());
                detener();
                JOptionPane.showMessageDialog(galy, "CREDITOS: \n miembros del equipo: el de los chescos: Soto Guzmán Moisés Nain \n la que editó las fotos: Maria Fernanda Rodriguez Rivas \n el creativo detrás de la historia: Ruben Aguilar Nuñez Alexis");
                global.resetInstancia();
                resetGaleryGui();
            }
            try {
                Thread.sleep(8000);
            } catch (InterruptedException ex) {
                //Thread.currentThread().interrupt();
                System.out.println(ex);
            }
        }
    }
````

Y así por medio de esto, ya manipulabas con creatividad la velocidad de la galería, porque sabes cuantos hilos ocuparas, en el tamagochi era más dinámico, pero sin chatgpt era un rollo
saber cómo podías detener un hilo o descubrir que las referencias de un objeto son transferibles entre métodos también, singleton... ¿Ahora ya visualizaste
cómo en que escenarios aplican estos hilos? es una concurrencia básica. Pero también caes en riesgo de tener código muy caótico. 

No voy a refactorizar esos códigos, que flojera, pero para que se den una idea de esos casos de uso de hilos generados manualmente.

La clase moderna es ExecutorService. 
