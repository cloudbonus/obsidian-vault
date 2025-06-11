```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierExample 
{
    private static CyclicBarrier FerryBarrier;
    private static final int FerryBoat_size = 3;

    // Переправляющий автомобили паром
    public static class FerryBoat implements Runnable 
    {
        @Override
        public void run() {
            try {
                // Задержка на переправе System.out.println(
                          "\nЗагрузка автомобилей");
                Thread.sleep(500);
                System.out.println(
                          "Паром переправил автомобили\n");
            } catch (InterruptedException e) {}
        }
    }

    // Класс автомобиля
    public static class Car implements Runnable
    {
        private int carNumber;

        public Car(int carNumber) {
            this.carNumber = carNumber;
        }

        @Override
        public void run() {
            try {
                System.out.printf(
                   "К переправе подъехал автомобиль %d\n",
                                               carNumber);
                // Вызов метода await при подходе к // барьеру; поток блокируется в ожидании // прихода остальных потоков
                FerryBarrier.await();
                System.out.printf(
                      "Автомобиль %d продолжил движение\n",
                                                carNumber);
            } catch (Exception e) {}
        }
    }

    public static void main(String[] args) 
                            throws InterruptedException {
        FerryBarrier = new CyclicBarrier(FerryBoat_size,
                                         new FerryBoat());
        for (int i = 1; i < 10; i++) {
            new Thread(new Car(i)).start();
            Thread.sleep(400);
        }
    }
}
```

**Результат выполнения примера**
Варьируя временем на переправе и временем прихода автомобилей на переправу, можно либо заставить паром простаивать, либо будут простаивать автомобили на переправе.

```bash
К переправе подъехал автомобиль 1
К переправе подъехал автомобиль 2
К переправе подъехал автомобиль 3

Загрузка автомобилей
К переправе подъехал автомобиль 4
Паром переправил автомобили

Автомобиль 3 продолжил движение
Автомобиль 1 продолжил движение
Автомобиль 2 продолжил движение
К переправе подъехал автомобиль 5
К переправе подъехал автомобиль 6

Загрузка автомобилей
К переправе подъехал автомобиль 7
Паром переправил автомобили

Автомобиль 6 продолжил движение
Автомобиль 5 продолжил движение
Автомобиль 4 продолжил движение
К переправе подъехал автомобиль 8
К переправе подъехал автомобиль 9

Загрузка автомобилей
Паром переправил автомобили

Автомобиль 7 продолжил движение
Автомобиль 9 продолжил движение
Автомобиль 8 продолжил движение
```