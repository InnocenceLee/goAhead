实现LRU过期淘汰策略：

模拟LinkedList实现内部定一个Node,然后包装成链表。

```java
public class CacheUtil<K, V> {

    private int capcity;

    private HashMap<K, Node<K, V>> hashMap;

    private MyLinkedList<K, V> list;

    public static void main(String[] args) {
        /**
         * LRU实现
         */
        CacheUtil<String, Integer> util = new CacheUtil<>(3);
        util.add("a", 1);
        util.show();
        util.add("b", 2);
        util.show();
        util.add("c", 3);
        util.show();
        util.add("d", 4);
        util.show();
        util.get("c");
        util.show();
        util.get("b");
        util.show();
    }

    public CacheUtil(int capacity) {
        this.capcity = capacity;
        hashMap = new HashMap<>(capacity);
        list = new MyLinkedList<>();
    }

    private void add(K key, V value) {
        Node node = new Node<>(key, value);
        if (hashMap.size() >= capcity) {
            list.remove(list.head.next);
        }
        hashMap.put(key, node);
        list.addLast(node);
    }

    private V get(K key) {
        if (!hashMap.containsKey(key)) {
            return null;
        } else {
            Node<K, V> node = hashMap.get(key);
            list.remove(node);
            list.addLast(node);
            return node.value;
        }
    }

    public void show() {
        list.show();
    }

    class Node<K, V> {
        K key;
        V value;
        Node<K, V> pre;
        Node<K, V> next;

        public Node() {
        }

        public Node(K k, V v) {
            this.key = k;
            this.value = v;
        }
    }

    class MyLinkedList<K, V> {
        Node<K, V> head;
        Node<K, V> tail;

        public MyLinkedList() {
            Node node = new Node();
            head = tail = node;
        }

        public void addLast(Node node) {
            tail.next = node;
            node.pre = tail;
            tail = node;
        }

        public void remove(Node node) {
            node.pre.next = node.next;
            node.next.pre = node.pre;
            node.pre = null;
            node.next = null;
        }

        public void show() {
            Node item = head;
            while (item.next != null) {
                System.out.print(item.next.value + "\t");
                item = item.next;
            }
            System.out.println();
        }
    }

}
```

