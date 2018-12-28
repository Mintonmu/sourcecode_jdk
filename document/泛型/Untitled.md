泛型=广泛的类型；就是类型参数化，处理的数据类型不是固定的，而是可以作为参数传入，同一套代码可以适用于多种数据类型，降低耦合、提高代码可读性、安全性，有泛型接口、类、方法。
一、类型参数的限定

1、上界为某个具体类
public class NumberPair<U extends Number,V extends Number> extends Pair<U,V>{
  public NumberPair(U first,V second){
       super(first,second);
  }
}

限定类型后，如果类型使用错误，编译器会提示，指定边界后，类型擦除时就不会转换为Object了，而是会转换为它的边界类型。

2、上界为某个接口
public static <T extends Comparable<T>> T max(T[] arr){
  T max = arr[0];
  for(int i=1;i<arr.length;i++){
   if(arr[i].compareTo(max) > 0){
     max = arr[i];
    }
   }
  return max;
}
<T extends Comparable<T>>是一种令人费解的语法形式，这种形式称为递归类型限制。
	
3、上界为其他类型参数
  public <T extends E> void addAll(DynamicArray<T> c) {
     for(int i=0; i<c.size; i++){
          add(c.get(i));
        }
  }
E是DynamicArray的类型参数，T是addAll的类型参数，T的上界限定为E，这样下面的代码就没有问题了
 DynamicArray<Number> numbers = new DynamicArray<>();
  DynamicArray<Integer> ints = new DynamicArray<>();
  ints.add(100);
  ints.add(34);
  numbers.addAll(ints);
  
  二、解析通配符
  1、更简洁的参数类型限定
  上面将Integer对象添加到Number容器中，我们的类型参数使用了其他类型参数作为上界，这种写法有点繁琐，他可以替换为更为简洁的通配符形式：
  public void addAll(DynamicArray<? extends E> c){
     for(int i=0; i<c.size; i++){
        add(c.get(i));
     }
  }
  1) <T extends E> 用于定义类型参数，它声明了一个类型参数T，可放在泛型类定义中类名后面、泛型方法返回值前面。
  2）<? extends E> 用于实例化类型参数，它用于实例化泛型变量中的类型参数，只是这个具体类型是未知的，只知道它是E或E的某个子类型。
两种写法：
public void addAll(DynamicArray<? extends E> c)
public <T extends E> void addAll(DynamicArray<T> c)

2、理解通配符
除了有限定通配符，还有一种通配符，形如DynamicArray<?>，称为无限定通配符，
其实这种无限定通配符形式也可以改为使用类型参数。
public static int indexOf（DynamicArray<?> arr,Object elm）
可以改为：
public static <T> int indexOf(DynamicArray<T> arr,Object elm)
但通配符有一个重要限制：只能读，不能写。

总结两种形式：
1）通配符形式都可以用类型参数的形式来替代，通配符能做的，用类型参数都能做。
2）通配符形式可以减少类型参数，形式上往往更为简单，可读性也更好，所以，能用通配符的就用通配符。
3）如果类型参数之间有依赖关系，或者返回值依赖类型参数，或者需要写操作，则只能用类型参数。
4）通配符形式和类型参数往往配合使用，比如，上面的copy方法，定义必要的类型参数，使用通配符表达依赖，并接受更广泛的数据类型。
private static <T> void swapInternal(DynamicArray<T> arr,int i,int j){
  T tmp = arr.get(i);
  arr.set(i,arr.get(j));
  arr.set(j,tmp);
  }
public static void swap(DynamicArray<?> arr,int i,int j){
  swapInternal(arr,i,j);
}
注：swap可以调用swapInternal，而带类型参数的swapInternal可以写入。java容器类中就有类似这样的用法，公共API是通配符形式，形式更简单，但内部调用带类型参数的方法。
public static <D> void copy(DynamicArray<D> dest,DynamicArray<? extends D> src){
   for(int i=0;i<src.size();i++){
    dest.add(src.get(i)); //dest是类型参数可以写入
}
}

3、超类型通配符
<? super E> 称为超类型通配符，表示E的某个父类型。
介绍了泛型中的三种通配符形式<?>、<? super E>和<? extends E>,并分析了与类型参数形式的区别和联系，总结比较如下：
1）它们的目的都是为了使方法接口更为灵活，可以接受更为广泛的类型。
2)<? super E>用于灵活写入或比较，使得对象可以写入父类型的容器，使得父类型的比较方法可以应用于子类对象，他不能被类型参数形式替代。
3）<?>和<? extends E>用于灵活读取，使得方法可以读取E或E的任意子类型的容器对象，它们可以用类型参数的形式替代，但通配符形式更为简洁。
