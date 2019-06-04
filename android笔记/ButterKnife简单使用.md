[TOC]

# ButterKnife简单使用

* 其实就是一个简化某部分代码编写工具；
* 我们总是会写大量的findViewById和点击事件，像初始view、设置view监听这样简单而重复的操作让人觉得特别麻烦，就是主要为了简化这部分的代码；

## 绑定View和点击事件及相关注意

1. ~~~java
   //如果只是在 app 中（而不是 lib 包）使用 ButterKnife ，就直接在 app 下的bulid.gradle中添加即可
   implementation 'com.jakewharton:butterknife:8.8.1'
   annotationProcessor 'com.jakewharton:butterknife-compiler:8.8.1'
   ~~~

2. 注意事项

   ~~~java
   setContentView(R.layout.activity_main);
   //一定要在setContentView 方法调用后调用 Butter Knife.bind：开始绑定
   ButterKnife.bind(this);
   ~~~

3. 绑定变量@BindView，@BindViews

   ~~~java
   //变量不可以被 private 和 static 修饰 
   @BindView(R.id.books)
    TextView mTextView;
   @BindView(R.id.textview)
    TextView mBookCount;

   @BindViews({R.id.books, R.id.textview})
   List<TextView> Views;
   ~~~

4. 绑定变量的点击事件@OnClick

   ~~~java
   //多个View就列在花括号内
   @OnClick({R.id.textview, R.id.books})
       public void onViewClicked(View view) {
           switch (view.getId()) {
               case R.id.textview:
                   processAddBook();
                   break;
               case R.id.books:
                   processGetBookCount();
                   break;
           }
   }
   ~~~

5. 为View指定动作：apply函数；也可以给 View的集合 设置统一的动作

   ~~~java
   static final ButterKnife.Action<View> VISITABLE = new ButterKnife.Action<View>() {
           @Override
           public void apply(@NonNull View view, int index) {
               view.setVisibility(View.VISIBLE);
           }
   };
   //调用
   ButterKnife.apply(Views,VISITABLE);
   ~~~

6. 在非 Activity 绑定 View，例如在 RecycleView 或 Fragment 中

   ~~~java
   //如在 RecycleView 中绑定View
   static class ViewHolder {
     @BindView(R.id.textview)
     TextView textview;
     @BindView(R.id.books) 
     TextView books;

     public ViewHolder(View view) {
       //简化了一大堆编写代码，直接交给 ButterKnife 即可；注意要把上下文给它
        ButterKnife.bind(this, view);
     }
   }
   ~~~

7. 在 Fragment 中需要注意，由于生命周期特殊，我们需要在 onDestroyView 把view设置为null。调用unbinder.unbind()方法即可；

   ~~~java
   //unbinder对象由 ButterKnife.bind 返回
   unbinder = ButterKnife.bind(this, view);

   @Override public void onDestroyView() {
           super.onDestroyView();
           unbinder.unbind();
   }
   ~~~

## 使用Zelezny插件再来简化 Butter Knife

* File/setting/Plugins/搜索Zelezny即可
* 在 View 的 上层 ViewGroup 如：R.layout.activity_main；点击右键/Generate/Generate Butter KnifeInjections即可



