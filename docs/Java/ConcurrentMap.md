__ConcurrentMap__

The `Hashtable` type in Java is a key value store that is thread-safe to use (in contrast to a `HashMap`). 
Nevertheless, `Hashtable` is often associated with performance issues because of its thread-safe guarantees.
For example, `Hashtable`s do not support concurrent reads or concurrent reads from and writes to different keys. 
Concrete types implementing the `ConcurrentMap` interface represents a key value store that is not only thread-safe to use, but also addresses 
performance limitations with a `Hashtable`.

```Java
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Iterator;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;

//TIP To <b>Run</b> code, press <shortcut actionId="Run"/> or
// click the <icon src="AllIcons.Actions.Execute"/> icon in the gutter.
public class Main {
    public static void main(String[] args) {
        // HashMap is not thread-safe
        Map<String, String> hashMap = new HashMap<>();
        // Hashtable is thread-safe but not performant
        Map<String, String> hashtable = new Hashtable<>();
        // ConcurrentHashMap is thread-safe & performant
        ConcurrentMap<String, String> concurrentMap = new ConcurrentHashMap<>();

        // Read/write operations
        concurrentMap.put("key", "value");
        concurrentMap.get("key");
        concurrentMap.remove("key");

        // Iterating through keys
        Iterator<String> iterator = concurrentMap.keySet().iterator();
        while (iterator.hasNext()) {
            System.out.println(iterator.next());
        }

        // Slipped condition
        if (!concurrentMap.containsKey("key")) {
            // Thread may context switch between the read & write
            concurrentMap.put("key", "value");
        }

        // Making the read/write atomic to address slipped condition
        concurrentMap.putIfAbsent("key", "value");

        // Making the read/write atomic if value to write depends on computation on the key
        concurrentMap.computeIfAbsent("key", k -> "value");

        // Making the read/write atomic if value to write depends on computation on the key/value
        concurrentMap.computeIfPresent("key", (key, value) -> value);
    }
}
```