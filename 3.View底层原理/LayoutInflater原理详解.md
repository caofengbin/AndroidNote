# LayoutInflater原理详解

---

## 1.由LayoutInflater谈起

``` java
layoutInflater.inflate(int resource, ViewGroup root) 
layoutInflater.inflate(int resource, ViewGroup root, boolean attachToRoot) 
```

&emsp;&emsp;官方文档关于LayoutInflater的描述：

``` java
Instantiates a layout XML file into its corresponding View objects. It is never used directly. Instead, use getLayoutInflater() or getSystemService(Class) to retrieve a standard LayoutInflater instance that is already hooked up to the current context and correctly configured for the device you are running on.
```

&emsp;&emsp;也就是说，LayoutInflater主要用来加载布局，而在setContentView()里，最终也是用到了LayoutInflater。

&emsp;&emsp;获取LayoutInflater的三种主要的方式：

> * LayoutInflater layoutInflater = LayoutInflater.from(context);
> * LayoutInflater layoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
> * LayoutInflater layoutInflater = Activity.getLayoutInflater();(第三种是要在activity里才有对应方法);

&emsp;&emsp;其中最常见的两种用法：

``` java
layoutInflater.inflate(int resource, ViewGroup root) 
layoutInflater.inflate(int resource, ViewGroup root, boolean attachToRoot) 
```

&emsp;&emsp;当时看到这个用法，头是很大的啊，第一个参数是传的xml的id，这都没问题，**最让人害怕的，就是后2个参数，到底是嘛意思嘛**！

---

## 2.源码分析

&emsp;&emsp;第一种方式的源码：

``` java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
    return inflate(resource, root, root != null);
}
```

&emsp;&emsp;其实就是调用了第二种方式：

> * 如果root == null，就相当于inflate(id,root,false);
> * 如果root != null，就相当于inflate(id,root,true);

&emsp;&emsp;看看第二种方式的调用源码：

``` java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }

```

&emsp;&emsp;这里提到了inflate(parser, root, attachToRoot)：

``` java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");

            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            
            1.注意啦！！！非常核心的一步，先把要返回的view用root来赋值，
            View result = root;

            try {
                // Look for the root node.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }

                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }

                final String name = parser.getName();
                
                if (DEBUG) {
                    System.out.println("**************************");
                    System.out.println("Creating root view: "
                            + name);
                    System.out.println("**************************");
                }

                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }

                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                
                    2. 注意啦！！！下面的createViewFromTag会返回xml布局里的最外层view。
                    // Temp is the root view that was found in the xml
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);  

                    ViewGroup.LayoutParams params = null;

                    if (root != null) {
                        if (DEBUG) {
                            System.out.println("Creating params from root: " +
                                    root);
                        }
                        
                        3. 注意啦！！！如果传的root不为null的时候，得到布局参数。
                        // Create layout params that match root, if supplied
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                        
                        4.注意啦！！！不被绑定的时候，而root又不为空，会把xml的view的参数设置
                            // Set the layout params for temp if we are not
                            // attaching. (If we are, we use addView, below)
                            temp.setLayoutParams(params);
                        }
                    }

                    if (DEBUG) {
                        System.out.println("-----> start inflating children");
                    }

                    // Inflate all children under temp against its context.
                    rInflateChildren(parser, temp, attrs, true);

                    if (DEBUG) {
                        System.out.println("-----> done inflating children");
                    }

                    // We are supposed to attach all the views we found (int temp)
                    // to root. Do that now.
                    if (root != null && attachToRoot) {
                    5.注意啦！！！ 如果root不为空，且第三个参数为true的时候，把这个xml渲染得到view作为子view加到root里去。
                        root.addView(temp, params);
                    }

                    // Decide whether to return the root that was passed in or the
                    // top view found in xml.
                    if (root == null || !attachToRoot) {
                    
                    6.注意啦！！！当root为空，或者（是或者哟！）第三个参数为空的时候，把返回的view用xml的view来赋值，
                    注意标注点1的时候，是一进来就把result用root来赋值哟。
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }

            Trace.traceEnd(Trace.TRACE_TAG_VIEW);

            return result;
        }

```

总结为流程图：

![image_1c4u3e219q8111u4kc9191n1tjtg.png-37.9kB][流程图]

相关的参数描述：

关于resource所渲染的view：

root条件 | attachRoot条件 | 结果
---- | --- | ---
root == null| attachRoot==true | 直接加载指定的布局文件(这里布局文件的宽高是没有用的)
root == null| attachRoot==false | 直接加载指定的布局文件(这里布局文件的宽高是没有用的)
root != null| attachRoot==true | 通过addView的方式添加
root != null| attachRoot==false | 使用xml文件的布局参数

关于返回值问题：

条件表达式 | 返回值
---- | ---
root == null 或者 attachRoot==false | resource所渲染的最外层根view
root非空 且 attachRoot==true | 根view--即RootView


参考链接：

[Android LayoutInflater列传](https://www.jianshu.com/p/1a327be3c6f3?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation)

[探究Android View 绘制流程，Xml 文件到 View 对象的转换过程](https://www.jianshu.com/p/eccd8ba87e8b)


  [流程图]: http://static.zybuluo.com/caofengbin/g8dnfbtoc2ohczrb2q67cjk9/image_1c4u3e219q8111u4kc9191n1tjtg.png