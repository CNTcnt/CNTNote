### RecycleView

* RecycleView是一个滚动布局控件


* 使用前要在build.gradle的dependencies闭包中添加依赖库

  > ```java
  > compile 'com.android.support:recyclerview-v7:24.2.1'
  > ```

* 先要把需要滚动的数据封装成一个类，即子项数据

  > ```java
  > public class contacts {//contacts:联系人
  >     String contacts_name;
  >     String contacts_number;
  >
  >     public void setContacts_name(String name ){
  >         contacts_name = name;}
  >     public String getContacts_name(){
  >         return contacts_name;}
  >     public void setContacts_number(String number){
  >         contacts_number = number;}
  >     public String getContacts_number(){
  >         return contacts_number;}}
  > ```

* 然后创建子项的布局，即装载要滚动的数据的布局

* 接下来创建适配器Adaper，继承于RecycleView.Adaper<适配器.场景持有器>，这个类中写下至少写下一个构造函数和一个数据列表成员，还必须复写3个方法。

  > ```java
  > List<contacts> contactsList;//这是数据列表成员
  > //这是构造函数，用于初始化数据数据列表成员
  >  public contactsAdaper(List<contacts> contactses){
  >         contactsList = contactses;
  >     }
  > ```

  > ```java
  > //方法一：这个方法是为了获取场景持有器
  > public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
  >   //先得到子项布局，先用布局筛选器加载到子项布局
  >     View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.item,parent,false);
  >   //再创建场景持有器，即内部类对象
  >     ViewHolder viewHolder = new ViewHolder(view);
  >     return viewHolder;
  > }
  > ```

  > ```java
  > //方法二：这个方法是为了向子项的数据进行赋值，在这里也可以做动画，因为可以拿到View的引用
  > public void onBindViewHolder(ViewHolder holder, int position) {
  >   //先利用position拿到当前滚动到屏幕内的子项数据
  >     contacts contacts = contactsList.get(position);
  >   //拿到子项数据后就可以给当前子项布局内的控件赋值了，使用场景持有器操作
  >     holder.name.setText(contacts.getContacts_name());
  >     holder.number.setText(contacts.getContacts_number());
  > }
  > ```

  > ```java
  > //方法三：这个方法用于告诉RecycleView有多少个子项
  > public int getItemCount() {
  >     return contactsList.size();
  > }//这个方法用于告诉RecycleView有多少个子项
  > ```

* 场景持有器ViewHolder，是适配器Adaper的静态内部类，继承于RecycleView.ViewHolder，控制子项布局的控件。

  > ```java
  > static class ViewHolder extends RecyclerView.ViewHolder{
  >     TextView name ;//创建子项布局内各个控件的引用，然后用view.findViewById（R.id.）来寻找子     TextView number;//项的各个控件
  >     public ViewHolder(View view){
  >         super(view);
  >         name =(TextView) view.findViewById(R.id.item_name);
  >         number = (TextView) view.findViewById(R.id.item_number);
  >     }
  > }
  > ```

* 在活动中向RecycleView添加数据,先在活动布局中找到RecyclerView控件，然后利用LinearLayoutManager来为RecycleView设置布局方式,然后再利用数据初始化出一个适配器，再把适配器放进RecycleView即可。

  ~~~java
  List<contacts>  contactsList = new ArrayList<>();//创建数据列表，数据是要初始化导入的，这里												//没有写出来
  RecyclerView recyclerView = (RecyclerView)findViewById(R.id.recyclerView);
  LinearLayoutManager layoutManager = new LinearLayoutManager(this);
  recyclerView.setLayoutManager(layoutManager.VERTICAL);//这里让RecycleView的子项竖直排列
  contactsAdaper contactsAdaper = new contactsAdaper(contactsList);
  recyclerView.setAdapter(contactsAdaper);
  ~~~
  ​

* ```
  //实现子项拖拽；
  ItemTouchHelper helper = new ItemTouchHelper(new ItemTouchHelper.Callback() {
      @Override
      public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
          int dragFlags = ItemTouchHelper.START | ItemTouchHelper.END;
          int swipeFlags = ItemTouchHelper.UP | ItemTouchHelper.DOWN;
          return makeMovementFlags(dragFlags, swipeFlags);
      }

      @Override
      public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
          int from=viewHolder.getAdapterPosition();
          int to=target.getAdapterPosition();
          Collections.swap(contactsList,from,to);
          contactsAdaper.notifyItemMoved(from, to);
          return true;
      }

      @Override
      public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
          int position=viewHolder.getAdapterPosition();
          contactsList.remove(position);
          contactsAdaper.notifyItemRemoved(position);
      }
      @Override
      public boolean isItemViewSwipeEnabled() {
          return true;
      }
      @Override
      public boolean isLongPressDragEnabled() {
          return true;
      }
  });
  helper.attachToRecyclerView(recyclerView);
  ```

