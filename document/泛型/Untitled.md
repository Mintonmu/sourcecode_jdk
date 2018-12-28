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
  
  
