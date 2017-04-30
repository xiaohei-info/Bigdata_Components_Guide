现有数据如下：
3 3   
3 2   
3 1   
2 2   
2 1   
1 1
要求为：
先按第一列从小到大排序，如果第一列相同，按第二列从小到大排序

如果是hadoop默认的排序方式，只能比较key，也就是第一列，而value是无法参与排序的
这时候就需要用到自定义的排序规则
解决思路：
自定义数据类型，将原本的key和value都包装进去
将这个数据类型当做key，这样就比较key的时候就可以包含第一列和第二列的值了

自定义数据类型NewK2如下：

```java
//要实现自定义的排序规则必须实现WritableComparable接口，泛型参数为类本身  
public class NewK2 implements WritableComparable<NewK2> {  
    //代表第一列和第二列的数据  
    Long first;  
    Long second;  

    public NewK2() {  
    }  

    public NewK2(long first, long second) {  
        this.first = first;  
        this.second = second;  
    }  
      
    //重写序列化和反序列化方法  
    @Override  
    public void readFields(DataInput in) throws IOException {  
        this.first = in.readLong();  
        this.second = in.readLong();  
    }  

    @Override  
    public void write(DataOutput out) throws IOException {  
        out.writeLong(first);  
        out.writeLong(second);  
    }  

    //当k2进行排序时，会自动调用该方法. 当第一列不同时，升序；当第一列相同时，第二列升序  
    //如果希望降序排列，那么只需要对调this对象和o对象的顺序  
    @Override  
    public int compareTo(NewK2 o) {  
        if(this.first != o.first)  
        {  
        return (int)(this.first - o.first);  
        }  
        else  
        {  
            return (int) (this.second - o.second);  
        }  
    }  
      
    //重写hashCode和equals方法  
    @Override  
    public int hashCode() {  
        return this.first.hashCode() + this.second.hashCode();  
    }  

    @Override  
    public boolean equals(Object obj) {  
        if (!(obj instanceof NewK2)) {  
            return false;  
        }  
        NewK2 oK2 = (NewK2) obj;  
        return (this.first == oK2.first) && (this.second == oK2.second);  
    }  
    @Override
    public String toString(){
        return this.first + '\t' + this.second;
    }
}
```

MyMapper类代码：

```java
public class MyMapper extends  
            Mapper<LongWritable, Text, NewK2, LongWritable> {  
        protected void map(  
                LongWritable key,  
                Text value,  
                org.apache.hadoop.mapreduce.Mapper<LongWritable, Text, NewK2, LongWritable>.Context context)  
                throws java.io.IOException, InterruptedException {  
            final String[] splited = value.toString().split("\t");  
            //分割完成之后的数据如：3,1  分别赋值给k2对象的first和second属性  
            final NewK2 k2 = new NewK2(Long.parseLong(splited[0]),  
                    Long.parseLong(splited[1]));  
            final LongWritable v2 = new LongWritable(Long.parseLong(splited[1]));  
            //将k2当做key输出，这样在排序的时候就会调用NewK2的compareTo方法，里面写的是我们自己的排序规则  
            context.write(k2, v2);  
        };  
    }
```

MyReducer类代码：

```java
public class MyReducer extends  
            Reducer<NewK2, LongWritable, LongWritable, LongWritable> {  
        protected void reduce(  
                NewK2 k2,  
                java.lang.Iterable<LongWritable> v2s,  
                org.apache.hadoop.mapreduce.Reducer<NewK2, LongWritable, LongWritable, LongWritable>.Context context)  
                throws java.io.IOException, InterruptedException {  
            context.write(new LongWritable(k2.first), new LongWritable(  
                    k2.second));  
        };  
    }
```

MySubmit类的代码和之前的一样无需改动
运行可得到结果如下图：

![图片](http://img.blog.csdn.net/20150211181629063?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXExMDEwODg1Njc4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

如果业务需求又发生了改变，如：上图结果中，第一列相同的，只要列出第二列的值最小的那个选项即可
那么结果应该为
1 1
2 1
3 1
可是我们之前使用的是自定义的数据类型当做key
而hadoop默认的分组策略是所有key相同的选项当做一组
而两个NewK2对象要相等，就必须要first和second属性都相等才行
这时就需要用到自定义的分组策略

自定义分组类如下：

```java
//自定义的分组类必须实现RawComparator，泛型参数为类本身  
    public class MyGroupingComparator implements RawComparator<NewK2> {  
        //重写两个比较方法  
          
        //按对象进行比较，规定只要两个NewK2对象的first属性相同就视为相等  
        @Override  
        public int compare(NewK2 o1, NewK2 o2) {  
            return (int) (o1.first - o2.first);  
        }  
  
        /** 
         * @param arg0 
         *            表示第一个参与比较的字节数组 
         * @param arg1 
         *            表示第一个参与比较的字节数组的起始位置 
         * @param arg2 
         *            表示第一个参与比较的字节数组的偏移量 
         *  
         * @param arg3 
         *            表示第二个参与比较的字节数组 
         * @param arg4 
         *            表示第二个参与比较的字节数组的起始位置 
         * @param arg5 
         *            表示第二个参与比较的字节数组的偏移量 
         */  
        @Override  
        public int compare(byte[] arg0, int arg1, int arg2, byte[] arg3,  
                int arg4, int arg5) {  
            return WritableComparator  
                    .compareBytes(arg0, arg1, 8, arg3, arg4, 8);  
        }  
  
    }
```

在MySubmit代码中加入设置分组策略

```java
// 1.4 TODO 排序、分区  
job.setGroupingComparatorClass(MyGroupingComparator.class);
```

再次运行程序可得到如下图的结果：

![图片](http://img.blog.csdn.net/20150211183156890?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXExMDEwODg1Njc4/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

作者：[@小黑](http://www.xiaohei.info)