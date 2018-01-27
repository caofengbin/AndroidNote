# Parcelable与Serializable原理

---

&emsp;&emsp;进行Android开发的时候，无法将对象的引用传给Activity或者Fragments，我们**需要将这些对象放到一个Intent或者Bundle里面，然后再传递。**
&emsp;&emsp;看api文档的时候，我们认识到有两种选择，我们的对象要么是Parcelable或者Serializable型，那么两种方式的区别在哪儿呢？

## 1.Serializable--简洁

``` java
// access modifiers, accessors and constructors omitted for brevity
public class SerializableDeveloper implements Serializable
    String name;
    int yearsOfExperience;
    List<Skill> skillSet;
    float favoriteFloat;

    static class Skill implements Serializable {
        String name;
        boolean programmingRelated;
    }
}
```

&emsp;&emsp;仅仅需要在它和它的子类上实现Serializable接口就能完成一个漂亮的Serializable功能，他是一个标记接口，意味着不需要实现任何方法，java虚拟机将简单高效地完成序列化工作。
&emsp;&emsp;这里面有个问题就是**这种序列化是通过反射机制从而削弱了性能，这种机制也创建了大量的临时对象从而引起GC频繁回收调用资源。**

---

## 2.Parcelable--速度

``` java
// access modifiers, accessors and regular constructors ommited for brevity
class ParcelableDeveloper implements Parcelable {
    String name;
    int yearsOfExperience;
    List<Skill> skillSet;
    float favoriteFloat;

    ParcelableDeveloper(Parcel in) {
        this.name = in.readString();
        this.yearsOfExperience = in.readInt();
        this.skillSet = new ArrayList<Skill>();
        in.readTypedList(skillSet, Skill.CREATOR);
        this.favoriteFloat = in.readFloat();
    }

    void writeToParcel(Parcel dest, int flags) {
        dest.writeString(name);
        dest.writeInt(yearsOfExperience);
        dest.writeTypedList(skillSet);
        dest.writeFloat(favoriteFloat);
    }

    int describeContents() {
        return 0;
    }


    static final Parcelable.Creator<ParcelableDeveloper> CREATOR
            = new Parcelable.Creator<ParcelableDeveloper>() {

        ParcelableDeveloper createFromParcel(Parcel in) {
            return new ParcelableDeveloper(in);
        }

        ParcelableDeveloper[] newArray(int size) {
            return new ParcelableDeveloper[size];
        }
    };

    static class Skill implements Parcelable {
        String name;
        boolean programmingRelated;

        Skill(Parcel in) {
            this.name = in.readString();
            this.programmingRelated = (in.readInt() == 1);
        }

        @Override
        void writeToParcel(Parcel dest, int flags) {
            dest.writeString(name);
            dest.writeInt(programmingRelated ? 1 : 0);
        }

        static final Parcelable.Creator<Skill> CREATOR
            = new Parcelable.Creator<Skill>() {

            Skill createFromParcel(Parcel in) {
                return new Skill(in);
            }

            Skill[] newArray(int size) {
                return new Skill[size];
            }
        };

        @Override
        int describeContents() {
            return 0;
        }
    }
}
```

&emsp;&emsp;按照google工程师的说话，这段代码将跑起来非常快，其中一个原因是**运用真实的序列化处理代替反射，为了完成这个目的代码也做了大量的优化。**
&emsp;&emsp;然而，显而易见的是实现Parcelable接口并不是无成本的，创建了大量的引入代码从而导致整个类变得很重同时加大了维护成本。推荐一款插件专门用来生成Parcelable相关的代码--[IntelliJ/Android Studio Plugin for Android Parcelable boilerplate code generation](https://github.com/mcharmas/android-parcelable-intellij-plugin)

---

## 3.性能测试

&emsp;&emsp;测试步骤 ：
>* 1：模拟这个操作通过Bundle的writeToParcel（Parcel, int）向Activity传递对象，然后观察它。
>* 2：循环这个操作1000次。 
>* 3：大概模拟10次，观察内存回收情况，以及app的cpu使用率，等等。 
>* 4：这个被测试的对象分别是SerializableDeveloper和ParcelableDeveloper。
>* 5：在多种机型和版本上做测试 LG Nexus 4 - Android 4.2.2 Samsung Nexus 10 - Android 4.2.2 HTC Desire Z - Android 2.3.3

![image_1c4ream7c1ld01vns1ubq6o01nl2m.png-16.6kB][1]

&emsp;&emsp;测试结果：

机型 | Serializable | Parcelable | 提升
---- | ---| ---| ---
Nexus 10 | 1.0004ms | 0.0850ms | 10.16倍
Nexus 4 |  1.8539ms | 0.1824ms | 11.80倍
Desire Z |  5.1224ms | 0.2938ms | 17.36倍


[原文地址：Parcelable vs Serializable](http://www.developerphil.com/parcelable-vs-serializable/)
  [1]: http://static.zybuluo.com/caofengbin/bpabuzifj6bv4eo99w60yapp/image_1c4ream7c1ld01vns1ubq6o01nl2m.png