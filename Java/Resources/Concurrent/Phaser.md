```java
import java.util.ArrayList;
import java.util.concurrent.Phaser;

public class PhaserExample
{
    private static Phaser PHASER;

    private static String OPEN  = "     открытие дверей ";
    private static String CLOSE = "     закрытие дверей ";

    public static void main(String[] args) 
                            throws InterruptedException
    {
        // Регистрация объекта синхронизации
        PHASER = new Phaser(1);

        ArrayList<Passenger> passengers;
        passengers = new ArrayList<>();
        // Формирование массива пассажиров
        for (int i = 1; i < 5; i++) {
            if ((int) (Math.random() * 2) > 0)
                // Этот пассажир проезжает одну станцию
                passengers.add(new Passenger(10 + i, i, 
                                                 i + 1));
            if ((int) (Math.random() * 2) > 0) {
                // Этот пассажир едет до конечной
                Passenger p = new Passenger(20 + i, i, 5);
               	passengers.add(p);
            }
        }

        // Фазы 0 и 6 - конечные станции // Фазы 1...5 - промежуточные станции
        for (int i = 0; i < 7; i++) {
            switch (i) {
                case 0:
                    System.out.println(
                                  "Метро вышло из тупика");
                    // Нулевая фаза, участников нет
                    PHASER.arrive();
                    break;
                case 6:
                    // Завершаем синхронизацию
                    System.out.println(
                                  "Метро ушло в тупик");
                    PHASER.arriveAndDeregister();
                    break;
                default:
                    int currentStation = PHASER.getPhase();
                    System.out.println(
                              "Станция " + currentStation);
                    // Проверка наличия пассажиров
                    // на станции
                    for (Passenger pass : passengers)
                        if (pass.departure==currentStation){
                            // Регистрация участника
                            PHASER.register();
                            new Thread (pass).start();
                        }
                    System.out.println(OPEN);

                    // Phaser ожидает завершения фазы // всеми участниками
                    PHASER.arriveAndAwaitAdvance();

                    System.out.println(CLOSE);
            }
        }
    }
}```

**Результат выполнения примера
Обратите внимание, что «всадник 3» завершил проверку, после чего «всадник 6» вошел в блокируемый участок кода. А вот «всадник 7» успел раньше вывести сообщение о входе в блокируемый участок кода, чем «всадник 1» сообщил о его «покидании».

```bash
Пассажир 11 {1 -> 2} ждёт на станции 1
Пассажир 21 {1 -> 5} ждёт на станции 1
Пассажир 12 {2 -> 3} ждёт на станции 2
Пассажир 23 {3 -> 5} ждёт на станции 3
Пассажир 14 {4 -> 5} ждёт на станции 4
Метро вышло из тупика
Станция 1
    ... открытие дверей ...
    Пассажир 11 {1 -> 2} вошел в вагон
    Пассажир 21 {1 -> 5} вошел в вагон
    ... закрытие дверей ...
Станция 2
    ... открытие дверей ...
    Пассажир 12 {2 -> 3} вошел в вагон
    Пассажир 11 {1 -> 2} вышел из вагона 
    ... закрытие дверей ...
Станция 3
    ... открытие дверей ...
    Пассажир 23 {3 -> 5} вошел в вагон
    Пассажир 12 {2 -> 3} вышел из вагона 
    ... закрытие дверей ...
Станция 4
    ... открытие дверей ...
    Пассажир 14 {4 -> 5} вошел в вагон
    ... закрытие дверей ...
Станция 5
    ... открытие дверей ...
    Пассажир 21 {1 -> 5} вышел из вагона 
    Пассажир 23 {3 -> 5} вышел из вагона 
    Пассажир 14 {4 -> 5} вышел из вагона 
    ... закрытие дверей ...
Метро ушло в тупик
```