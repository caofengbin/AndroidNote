# 深入理解Bundle和Map

---

## 1.案例：往Bundle对象放入特殊的Map

&emsp;&emsp;你需要将一个要传递的map附加到Intent对象。这个案例虽然不常见，但是，这种情况也是很有可能发生。

&emsp;&emsp;如果你**在Intent对象中附加的是一个Map最常见的接口实现类HashMap**，而不是包含附加信息的自定义类，你是幸运的，你可以用以下方法将map附加到Intent对象:

``` java
intent.putExtra("map",myHashMap);
```

&emsp;&emsp;在你接收的Activity里，你可以用以下方法毫无问题地取出之前在Intent中附加的Map:

``` java
 HashMap map = (HashMap) getIntent().getSerializableExtra("map");
```

&emsp;&emsp;但是，如果你在Intent对象附加另一种类型的Map，比如：一个TreeMap（或者其他的自定义Map接口实现类），你在Intent中取出之前附加的TreeMap时，你用如下方法：

``` java
 TreeMap map = (TreeMap) getIntent().getSerializableExtra("map");`
```

&emsp;&emsp;然后就会出现一个类转换异常:

``` java
java.lang.ClassCastException: java.util.HashMap cannot be cast to java.util.TreeMap`
```

&emsp;&emsp;**因为编译器认为你的Map(TreeMap)正试图转换成一个HashMap!!!**

---

## 2.为什么用getSerializableExtra()方法来取出附加到Intent中的Map

&emsp;&emsp;因为所有默认的 Map 接口实现类都是Serializable,并且 putExtra()/getExtra() 方法接受的参数几乎都是“键-值”对，而其中**值的类型非常广泛，Serializable就是其中之一，因此我们能够使用 getSerializableExtra() 来取得我们传递的Map。**

---

## 3.关于Parcels

&emsp;&emsp;Parcel是Android进程间通信中， 高效的专用序列化机制。

&emsp;&emsp;A Parcel is an optimised, non-general purpose serialisation mechanism that Android employs for IPC. Contrary to Serializable objects, you should never use Parcels for any kind of persistence, as it does not provision for handling different versions of the data. Whenever you see a Bundle, you’re dealing with a Parcel under the hood.

&emsp;&emsp;Parcels know how to handle a bunch of types out of the box, including native types, strings, arrays, maps, sparse arrays, parcelables and serializables. Parcelables are the mechanism that you (should) use to write and read arbitrary data to a Parcel, unless you really, really need to use Serializable.

&emsp;&emsp;The advantages of Parcelable over Serializable are mostly about performances, and that should be enough of a reason to prefer the former in most cases, as Serializable comes with a certain overhead.

---

## 4.深入底层分析

&emsp;&emsp;让我们来了解下是什么原因使我们得到了ClassCastException异常。 从我们的代码中可以看到，我们对Intent中putExtras()的调用实际上是传入了一个String值和一个Serializable的对象，而不是传入一个Map值。因为Map接口实现类都是Serializable的，而不是Parcelable的。

### 第一步：找到第一个突破口

&emsp;&emsp;让我们来看看在Intent.putExtra(String, Serializable)方法中做了什么。

``` java

Intent.java

public Intent putExtra(String name, Serializable value) {
      // ...
      mExtras.putSerializable(name, value);
      return this;
    }
}

```

&emsp;&emsp;在这里，mExtras是个Bundle，Intent指令所有附加信息到bundle，从而调用了Bundle的putSerializable()方法，让我们来看看在Bundle中的putSerializable()方法中做了什么：

``` java
Bundle.java

public void putSerializable(String key, Serializable value) {
      super.putSerializable(key, value);
}
```

&emsp;&emsp;从上面代码我们可以看出，Bundle中的putSerializable()方法中只是对父类的实现，调用了父类BaseBundle中的putSerializable（）方法。

``` java
void putSerializable(String key, Serializable value) {
      unparcel();
      mMap.put(key, value);
}

 ArrayMap<String, Object> mMap = null;
 
```

&emsp;&emsp;首先，让我们忽略其中的unparcel()这个方法。我们注意到mMap是一个ArrayMap<String, Object>类型的。这告诉我们，到了这步，我们往mMap中设入的是一个Object类型的。也就是说，**不管我们之前是什么类型，在父类BaseBundle这里都转成了Obeject类型。**

### 第二步：分析写入 map

&emsp;&emsp;有趣的是当把Bundle中的值写入到一个Parcel中时，如果此时我们去检查我们附加值的类型，我们发现仍然能得到正确的类型。

``` java
Intent intent = new Intent(this, ReceiverActivity.class);
intent.putExtra("map", treeMap);
Serializable map = intent.getSerializableExtra("map");
Log.i("MAP TYPE", map.getClass().getSimpleName());
```

&emsp;&emsp;如我们所料，这里打印出来的是TreeMap类型的。因此，在Bundle中写成一个Parcel，与再次读这期间一定发生了类型转换。

&emsp;&emsp;如果我们观察下是怎样写入Parcel的，我们看到，实际上是调BaseBundle中的writeToParcelInner()方法。

``` java
void writeToParcelInner(Parcel parcel, int flags) {
  if (mParcelledData != null) {
    // ...
  } else {
    // ...
    int startPos = parcel.dataPosition();
    parcel.writeArrayMapInternal(mMap);
    int endPos = parcel.dataPosition();
    // ...
  }
}

```

&emsp;&emsp;跳过所有不相干的代码，我们看到在Parcel的writeArrayMapInternal()方法中做了大量的事（mMap 是一个 ArrayMap类型）

``` java
Parcel.java

    void writeArrayMapInternal(
    	ArrayMap<String, Object> val) {
      	// ...
      	int startPos;
      	for (int i=0; i<N; i++) {
    	// ...
    	writeString(val.keyAt(i));
    	writeValue(val.valueAt(i));
    	// ...
      }
    }
```

###第三步：分析写入Map值

``` java
Parcel.java

public final void writeValue(Object v) {
	      if (v == null) {
	    	writeInt(VAL_NULL);
	      } else if (v instanceof String) {
		    writeInt(VAL_STRING);
		    writeString((String) v);
	      } else if (v instanceof Integer) {
		    writeInt(VAL_INTEGER);
		    writeInt((Integer) v);
	      } else if (v instanceof Map) {
		    writeInt(VAL_MAP);
		    writeMap((Map) v);
	      } else if (/* you get the idea, this goes on and on */) {
	    	// ...
	      } else {
	    	Class<?> clazz = v.getClass();
	    	if (clazz.isArray() &&
	    	clazz.getComponentType() == Object.class) {
	      // Only pure Object[] are written here, Other arrays of non-primitive types are
	      // handled by serialization as this does not record the component type.
	      	writeInt(VAL_OBJECTARRAY);
	     	 writeArray((Object[]) v);
	    } else if (v instanceof Serializable) {
	      // Must be last
	      writeInt(VAL_SERIALIZABLE);
	      writeSerializable((Serializable) v);
	    } else {
	      throw new RuntimeException("Parcel: unable to marshal value "+ v);
	    }
	      }
	    }
```

&emsp;&emsp;**虽然TreeMap是以Serializable的类型传入到 bundle，但是在Parcel中writeValue(）方法执行的是map这个分支的代码**---“v instanceof Map”，（“v instanceof Map”在“v instanceOf Serializable”之前）

###第四步：分析将Map写入到Parcel中

&emsp;&emsp;Parcel中的writeMap()方法并没有做什么事，只是将我们传入的Map值强转成Map<String, Object>类型，调用writeMapInternal（）方法。

``` java
Parcel.java

public final void writeMap(Map val) {	
    writeMapInternal((Map<String, Object>) val);
}

```

&emsp;&emsp;尽管我们可能传入一个key值不为String的Map,类型擦除也使我们不会获得运行时错误。(这是完全非法的)事实上，看一下Parcel中的writeMapInternal()方法，这更打击我们。

``` java
    void writeMapInternal(Map<String,Object> val) {
      // ...
      Set<Map.Entry<String,Object>> entries = val.entrySet();
      writeInt(entries.size());
      for (Map.Entry<String,Object> e : entries) {
    	writeValue(e.getKey());
    	writeValue(e.getValue());
      }
    }
```

&emsp;&emsp;**类型擦除使所有的这些代码都不会出现运行时错误。**


###第五步：分析读Map

&emsp;&emsp;让我们来看看Parcel中readValue()这个方法，这个方法和writeValue()相对应。

``` java
Parcel.java

public final Object readValue(ClassLoader loader) {
      int type = readInt();
    
      switch (type) {
    	case VAL_NULL:
      	return null;
    
    	case VAL_STRING:
      	return readString();
    
    	case VAL_INTEGER:
      	return readInt();
    
    	case VAL_MAP:
    	// 关键点在这里！！！！
     	return readHashMap(loader);
    
    // ...
      }
    }
```

&emsp;&emsp;这里我们可以看到，readValue()方法中，首先读取一个int的数据，这个int数据是在writeValue（）中将TreeMap设成的VAL_MAP的常量，**然后去匹配后面的分支,调用readHashMap()方法来取回数据。**

``` java
public final HashMap readHashMap(ClassLoader loader)
    {
      int N = readInt();
      if (N < 0) {
     return null;
      }
      HashMap m = new HashMap(N);
      readMapInternal(m, N, loader);
      return m;
    }
```

&emsp;&emsp;readMapInternal()这个方法只是将我们从Parcel中读取的map重新进行打包。

&emsp;&emsp;**这就是为什么我们总是从Bundle中获得一个HashMap，同样的，如果你创建了一个实现了Parcelable自定义类型Map,得到的也是一个HashMap。**

&emsp;&emsp;很难说本身设计如此，还是是一个疏忽。这确实是一个极端例子，**因为在一个Intent中传一个Map是比较少见的，你也只有很小的理由来传Serializable而不是Parcelable的。**

---

## 5.小结

&emsp;&emsp;We have the huge luxury (and curse) of having access to the AOSP code. That’s something almost unique in the mobile landscape. We can know to a certain extent exactly what goes on. And we should.

&emsp;&emsp;Because even though it might look like it’s WTF-land sometimes, you can only become a better developer when you get to know the inner workings of the platform you work on.

&emsp;&emsp;And remember: what doesn’t kill you makes you stronger. Or crazier.

---

&emsp;&emsp;[1.原文链接--The mysterious case of the Bundle and the Map](https://medium.com/the-wtf-files/the-mysterious-case-of-the-bundle-and-the-map-7b15279a794e)





