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


Le continuaremos mañana pero si está interesante.
