# CopyOnWriteArrayList

​		 我们平时开发用的比较多的是ArrayList，追溯源码知道它在多线程环境下是线程不安全的，通过一个示例演示一下。单线程或并发场景下可能会出现ConcurrentModificationException（但一般是针对并发场景），它

```java
public class ArrayListTest {

    public List<Integer> list = new ArrayList<>();

    @Test
    public void demo() throws InterruptedException {
        // 开启两条线程进行写操作
        for (int i = 0; i < 2; i ++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int i = 1; i <= 10; i ++) {
                        list.add(i);
                    }
                }
            }).start();
        }
        Thread.sleep(1l);
        System.out.println(list);
    }
}
```

![](C:\Users\13509\Desktop\java_learn\JavaLearn\photo\JavaSE\集合\Concurrent\ArrayList并发存在的问题.png)