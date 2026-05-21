```java
import java.util.HashMap;
import java.util.Iterator;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class HashMapExample 
{
    Map<String, String> map;

    public ConcurrentHashMapExample()
    {
        System.out.println("ConcurrentHashMap");
        createMap(true);
        addValue (true);

        System.out.println("\n\nHashMap");
        createMap(false);
        addValue (false);
    }

    private void addValue(boolean concurrent)
    {
        System.out.println("  before iterator : " + map);
        Iterator<String> t = map.keySet().iterator();

        System.out.print("  cycle : ");
        while(it.hasNext()){
            String key = it.next();
            if (key.equals("2")) {
                map.put(key + "new", "222");
            } else
                System.out.print("  " + key + "="
                                      + map.get(key));
        }
        System.out.println();
        System.out.println("  after iterator : " + map);
    }

    private void createMap(boolean concurrent)
    {
        if (concurrent)
            map = new ConcurrentHashMap<String, String>();
        else
            map = new HashMap<String, String> ();
        map.put("1", "1");
        map.put("2", "1");
        map.put("3", "1");
        map.put("4", "1");
        map.put("5", "1");
        map.put("6", "1");
    }

    public static void main(String[] args) 
    {
        new ConcurrentHashMapExample();
        System.exit(0);
    }
}
```

**Результат выполнения примера**
Информационные сообщения при выполнении примера выводятся в консоль. Что мы видим? При использование класса ConcurrentHashMap цикл перебора с использованием итератора завершился нормально; в консоль попал также новый объект с ключом "2", добавленный в набор во время итерации. А вот при использовании класса HashMap цикл был прерван вызовом исключения ConcurrentModificationException, как и ожидалось. Место ошибки : String key = it.next();. Т.е. итератор вызывает исключение при обращении к следующему объекту, если набор изменился.

```bash
ConcurrentHashMap
  before iterator : {1=1, 2=1, 3=1, 4=1, 5=1, 6=1}
  cycle :   1=1  3=1  4=1  5=1  6=1  2new=222
  after iterator : {1=1, 2=1, 3=1, 4=1, 5=1, 6=1, 2new=222}

HashMap
  before iterator : {1=1, 2=1, 3=1, 4=1, 5=1, 6=1}
  cycle :   1=1 \
Exception in thread "main" 
    java.util.ConcurrentModificationException
   at java.util.HashMap$HashIterator.nextNode(Unknown Source)
   at java.util.HashMap$KeyIterator.next(Unknown Source)
   at example.HashMapExample.addValue(HashMapExample.java:30)
   at example.HashMapExample.<init>(HashMapExample.java:20)
   at example.HashMapExample.main(HashMapExample.java:58)
```

Конечно, данную проблему с классом HashMap можно решить, дополнив код прерыванием цикла

```java
if (key.equals("2")) {
    map.put(key + "new", "222");
    if (!concurrent)
        break;
} else
    System.out.print("  " + key + "=" + map.get(key));
```