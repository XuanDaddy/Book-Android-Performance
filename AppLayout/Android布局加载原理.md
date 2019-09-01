# 【一】Android布局加载原理

### 布局加载流程（xml）

1. AppCompatActivity.setContentView(int resId)

   ```java
   @Override
   public void setContentView(@LayoutRes int layoutResID) {
      getDelegate().setContentView(layoutResID);
   }
   ```

2. AppCompatDelegate.setContentView(int resId):抽象类

   ```java
   public abstract void setContentView(@LayoutRes int resId);
   ```

3. AppCompatDelegateImpl.setContentView(int resId):实现类

   ```java
   @Override
   public void setContentView(int resId) {
      ensureSubDecor();
      ViewGroup contentParent = (ViewGroup) mSubDecor.findViewById(android.R.id.content);
      contentParent.removeAllViews();
      LayoutInflater.from(mContext).inflate(resId, contentParent);
      mOriginalWindowCallback.onContentChanged();
   }
   ```

   加载布局使用的LayoutInflater

4. LayoutInflater.inflate()

   ```java
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

   调用 res.getLayout()

5. Resourecs.getLayout()

   ```java
       public XmlResourceParser getLayout(@LayoutRes int id) throws NotFoundException {
           return loadXmlResourceParser(id, "layout");
       }
   
       XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type)
               throws NotFoundException {
           final TypedValue value = obtainTempTypedValue();
           try {
               final ResourcesImpl impl = mResourcesImpl;
               impl.getValue(id, value, true);
               if (value.type == TypedValue.TYPE_STRING) {
                   return impl.loadXmlResourceParser(value.string.toString(), id,
                           value.assetCookie, type);
               }
               throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                       + " type #0x" + Integer.toHexString(value.type) + " is not valid");
           } finally {
               releaseTempTypedValue(value);
           }
       }
   ```

   加载xml文件，从返回一个xml解析器。

   **从xml文件加载到内存中，这里其实是一个IO过程**。

6. LayoutInflater.createViewFromTag()

   ```java
     View createViewFromTag(View parent, String name, Context context, AttributeSet attrs,
               boolean ignoreThemeAttr) {
           if (name.equals("view")) {
               name = attrs.getAttributeValue(null, "class");
           }
   
           // Apply a theme wrapper, if allowed and one is specified.
           if (!ignoreThemeAttr) {
               final TypedArray ta = context.obtainStyledAttributes(attrs, ATTRS_THEME);
               final int themeResId = ta.getResourceId(0, 0);
               if (themeResId != 0) {
                   context = new ContextThemeWrapper(context, themeResId);
               }
               ta.recycle();
           }
   
           if (name.equals(TAG_1995)) {
               // Let's party like it's 1995!
               return new BlinkLayout(context, attrs);
           }
   
           try {
               View view;
               if (mFactory2 != null) {
                   view = mFactory2.onCreateView(parent, name, context, attrs);
               } else if (mFactory != null) {
                   view = mFactory.onCreateView(name, context, attrs);
               } else {
                   view = null;
               }
   
               if (view == null && mPrivateFactory != null) {
                   view = mPrivateFactory.onCreateView(parent, name, context, attrs);
               }
   
               if (view == null) {
                   final Object lastContext = mConstructorArgs[0];
                   mConstructorArgs[0] = context;
                   try {
                       if (-1 == name.indexOf('.')) {
                           view = onCreateView(parent, name, attrs);
                       } else {
                           view = createView(name, null, attrs);
                       }
                   } finally {
                       mConstructorArgs[0] = lastContext;
                   }
               }
   
               return view;
           } catch (InflateException e) {
               throw e;
   
           } catch (ClassNotFoundException e) {
               final InflateException ie = new InflateException(attrs.getPositionDescription()
                       + ": Error inflating class " + name, e);
               ie.setStackTrace(EMPTY_STACK_TRACE);
               throw ie;
   
           } catch (Exception e) {
               final InflateException ie = new InflateException(attrs.getPositionDescription()
                       + ": Error inflating class " + name, e);
               ie.setStackTrace(EMPTY_STACK_TRACE);
               throw ie;
           }
       }
   ```

   优先使用mFactory2进行创建，如果没有设置，使用mFactory进行创建；

   mPrivateFactory只会用于处理fragment标签；

   如果View依然为空，会调用LayoutInflater.createView通过反映进行创建；

   **反射也是一个性能瓶颈**

7. LayoutInflater.createView()

   ```java
   ...
    final View view = constructor.newInstance(args);
               if (view instanceof ViewStub) {
                   // Use the same context when inflating ViewStub later.
                   final ViewStub viewStub = (ViewStub) view;
                   viewStub.setLayoutInflater(cloneInContext((Context) args[0]));
               }
               mConstructorArgs[0] = lastContext;
               return view;
   ```

### LayoutInflater.Factory

1. 它是LayoutInflater创建View的一个Hook;
2. 它可以用来定制创建View的过程：全局替换自定义TextView等；
3. **Factory2**继承自**Factory**；

```java
    public interface Factory {
        /**
         * Hook you can supply that is called when inflating from a LayoutInflater.
         * You can use this to customize the tag names available in your XML
         * layout files.
         */
        public View onCreateView(String name, Context context, AttributeSet attrs);
    }
```

name对应的就是View名称，如"TextView"，"ImageView"等。

```java
    public interface Factory2 extends Factory {
        /**
         * Version of {@link #onCreateView(String, Context, AttributeSet)}
         * that also supplies the parent that the view created view will be
         * placed in.
         */
        public View onCreateView(View parent, String name, Context context, AttributeSet attrs);
    }
```

相比Fatory，Factory2中的onCreateView返回了parent，也就是创建的View的父布局。

### AppCompatActivity默认设置Factory

在Activity中并未设置factory，但在AppCompatActivity中onCreate默认设置了factory。

如果我们直接设置factory，会导致系统默认的factory无法使用。

```
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    getDelegate().installViewFactory();
    //...
}
```

```java
@Override
public void installViewFactory() {
    LayoutInflater layoutInflater = LayoutInflater.from(mContext);
    if (layoutInflater.getFactory() == null) {
        LayoutInflaterCompat.setFactory(layoutInflater, this);
    } else {
        Log.i(TAG, "The Activity's LayoutInflater already has a Factory installed"
                + " so we can not install AppCompat's");
    }
}
```

AppCompatActivity的setFactory也是想根据name去生成一些类，大家还记得，更新v7包的时候，忽然我们的TextView就支持了一些属性，比如`tint`属性，以前是不支持的，怎么能够做到使用v7包，然后就能支持且向下兼容的呢？

```java
switch (name) {
    case "TextView":
        view = new AppCompatTextView(context, attrs);
        break;
    case "ImageView":
        view = new AppCompatImageView(context, attrs);
        break;
    case "Button":
        view = new AppCompatButton(context, attrs);
        break;
    case "EditText":
        view = new AppCompatEditText(context, attrs);
        break;
   //...
}
```

可以看到系统其实是利用setFactory，瞒着我们把TextView等类早就换成AppCompatTextView等类了。

#### 如何解决自定义factory覆盖系统默认

我们可以在自定义Factory中处理自己的逻辑，默认的依然交给系统处理(delegate)。

自定义Factory需要在super.onCreate()之前进行配置。

```java
LayoutInflaterCompat.setFactory(LayoutInflater.from(this), new LayoutInflaterFactory()
{
    @Override
    public View onCreateView(View parent, String name, Context context, AttributeSet attrs)
    {
        //你可以在这里直接new自定义View

        //你可以在这里将系统类替换为自定义View

         //appcompat 创建view代码
        AppCompatDelegate delegate = getDelegate();
        View view = delegate.createView(parent, name, context, attrs);

        return view;
    }
});
```

### 总结

* 加载流程

setContentView加载xml文件，返回xml解析器，通过解析器判断是否存在factory，如果存在，使用factory构建View实例（AppcompatActivity已默认设置了factory，Activity并没有），如果未设置factory或View为空，使用反射构建View实例。

* 性能瓶颈
  * 加载xml文件，IO操作；
  * 如果未配置factory，反射构建View实例；