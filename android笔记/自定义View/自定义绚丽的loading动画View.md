 尊重原创，欢迎转载，转载请注明： FROM  GA_studio   [http://blog.csdn.net/tianjian4592](http://blog.csdn.net/tianjian4592)

       前两天我们这边的头儿给我说，有个 gif 动效很不错，可以考虑用来做项目里的loading，问我能不能实现，看了下效果确实不错，也还比较有新意，复杂度也不是非常高，所以就花时间给做了，我们先一起看下原gif图效果：

![](https://img-blog.csdn.net/20150322173842391?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGlhbmppYW40NTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

\--------------------------------------------------

如果你想看 [GAStudio Github主页](https://github.com/Ajian-studio)，请戳[这里](https://github.com/Ajian-studio)； 

如果你想看 [GAStudio Github主页](https://github.com/Ajian-studio)，请戳[这里](https://github.com/Ajian-studio)； 
如果你想看 [GAStudio](http://blog.csdn.net/tianjian4592)更多[技术文章](http://blog.csdn.net/tianjian4592)，请戳[这里](http://blog.csdn.net/tianjian4592)； 

如果你想看 [GAStudio Github主页](https://github.com/Ajian-studio)，请戳[这里](https://github.com/Ajian-studio)； 
如果你想看 [GAStudio](http://blog.csdn.net/tianjian4592)更多[技术文章](http://blog.csdn.net/tianjian4592)，请戳[这里](http://blog.csdn.net/tianjian4592)； 
QQ技术交流群：277582728； 

\--------------------------------------------------

从效果上看，我们需要考虑以下几个问题：

1.叶子的随机产生；

2.叶子随着一条正余弦曲线移动；

3.叶子在移动的时候旋转，旋转方向随机，正时针或逆时针；

4.叶子遇到进度条，似乎是融合进入；

5.叶子不能超出最左边的弧角；

7.叶子飘出时的角度不是一致，走的曲线的振幅也有差别，否则太有规律性，缺乏美感；

总的看起来，需要注意和麻烦的地方主要是以上几点，当然还有一些细节问题，比如最左边是圆弧等等；

那接下来我们将效果进行分解，然后逐个击破：

整个效果来说，我们需要的图主要是飞动的小叶子和右边旋转的风扇，其他的部分都可以用色值进行绘制，当然我们为了方便，就连底部框一起切了；

先从gif 图里把飞动的小叶子和右边旋转的风扇、底部框抠出来，小叶子图如下：

                                          ![](https://img-blog.csdn.net/20150322181205172?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGlhbmppYW40NTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

我们需要处理的主要有两个部分：

\1. 随着进度往前绘制的进度条；

\2. 不断飞出来的小叶片；

我们先处理第一部分 － 随着进度往前绘制的进度条：

进度条的位置根据外层传入的 progress 进行计算，可以分为图中 1、2、3 三个阶段：

![](https://img-blog.csdn.net/20150322215713127?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGlhbmppYW40NTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

\1. 当progress 较小，算出的当前距离还在弧形以内时，需要绘制如图所示 1 区域的弧形，其余部分用白色填充；

\2. 当 progress 算出的距离到2时，需要绘制棕色半圆弧形，其余部分用白色矩形填充；

\3. 当 progress 算出的距离到3 时，需要绘制棕色半圆弧形，棕色矩形，白色矩形；

\4. 当 progress 算出的距离到头时，需要绘制棕色半圆弧形，棕色矩形；（可以合并到3中）

首先根据进度条的宽度和当前进度、总进度算出当前的位置：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. //mProgressWidth为进度条的宽度，根据当前进度算出进度条的位置  
2. mCurrentProgressPosition = mProgressWidth * mProgress / TOTAL_PROGRESS;  

然后按照上面的逻辑进行绘制，其中需要计算上图中的红色弧角角度，计算方法如下：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. // 单边角度  
2. int angle = (int) Math.toDegrees(Math.acos((mArcRadius - mCurrentProgressPosition)/ (float) mArcRadius));  

Math.acos()  －反余弦函数；

Math.toDegrees() － 弧度转化为角度，Math.toRadians 角度转化为弧度

所以圆弧的起始点为：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. int startAngle = 180 - angle;  

圆弧划过的角度为：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. 2 * angle  

这一块的代码如下：

~~~java
// mProgressWidth为进度条的宽度，根据当前进度算出进度条的位置  
mCurrentProgressPosition = mProgressWidth * mProgress / TOTAL_PROGRESS;  
// 即当前位置在图中所示1范围内  
if (mCurrentProgressPosition < mArcRadius) {  
    Log.i(TAG, "mProgress = " + mProgress + "---mCurrentProgressPosition = "  
            + mCurrentProgressPosition  
            + "--mArcProgressWidth" + mArcRadius);  
    // 1.绘制白色ARC，绘制orange ARC  
    // 2.绘制白色矩形  
  
    // 1.绘制白色ARC  
    canvas.drawArc(mArcRectF, 90, 180, false, mWhitePaint);  
  
    // 2.绘制白色矩形  
    mWhiteRectF.left = mArcRightLocation;  
    canvas.drawRect(mWhiteRectF, mWhitePaint);  
  
    // 3.绘制棕色 ARC  
    // 单边角度  
    int angle = (int) Math.toDegrees(Math.acos((mArcRadius - mCurrentProgressPosition)  
            / (float) mArcRadius));  
    // 起始的位置  
    int startAngle = 180 - angle;  
    // 扫过的角度  
    int sweepAngle = 2 * angle;  
    Log.i(TAG, "startAngle = " + startAngle);  
    canvas.drawArc(mArcRectF, startAngle, sweepAngle, false, mOrangePaint);  
} else {  
    Log.i(TAG, "mProgress = " + mProgress + "---transfer-----mCurrentProgressPosition = "  
            + mCurrentProgressPosition  
            + "--mArcProgressWidth" + mArcRadius);  
    // 1.绘制white RECT  
    // 2.绘制Orange ARC  
    // 3.绘制orange RECT  
     
    // 1.绘制white RECT  
    mWhiteRectF.left = mCurrentProgressPosition;  
    canvas.drawRect(mWhiteRectF, mWhitePaint);  
      
    // 2.绘制Orange ARC  
    canvas.drawArc(mArcRectF, 90, 180, false, mOrangePaint);  
    // 3.绘制orange RECT  
    mOrangeRectF.left = mArcRightLocation;  
    mOrangeRectF.right = mCurrentProgressPosition;  
    canvas.drawRect(mOrangeRectF, mOrangePaint);  
  
}  
~~~



接下来再来看叶子部分：

首先根据效果情况基本确定出 曲线函数，标准函数方程为：y = A(wx+Q)+h，其中w影响周期，A影响振幅 ，周期T＝ 2 * Math.PI/w;

根据效果可以看出，周期大致为总进度长度，所以确定w＝(float) ((float) 2 * Math.PI /mProgressWidth)；

仔细观察效果，我们可以发现，叶子飘动的过程中振幅不是完全一致的，产生一种错落的效果，既然如此，我们给叶子定义一个Type，根据Type 确定不同的振幅；

我们创建一个叶子对象：

~~~java
private class Leaf {  
      // 在绘制部分的位置  
      float x, y;  
      // 控制叶子飘动的幅度  
      StartType type;  
     // 旋转角度  
      int rotateAngle;  
      // 旋转方向--0代表顺时针，1代表逆时针  
      int rotateDirection;  
      // 起始时间(ms)  
      long startTime;  
}  
~~~



类型采用枚举进行定义，其实就是用来区分不同的振幅：

~~~java
 private enum StartType {  
     LITTLE, MIDDLE, BIG  
   }  
~~~

创建一个LeafFactory类用于创建一个或多个叶子信息：

~~~java


private class LeafFactory {  
     private static final int MAX_LEAFS = 6;  
     Random random = new Random();  
   
     // 生成一个叶子信息  
     public Leaf generateLeaf() {  
         Leaf leaf = new Leaf();  
         int randomType = random.nextInt(3);  
         // 随时类型－ 随机振幅  
         StartType type = StartType.MIDDLE;  
         switch (randomType) {  
             case 0:  
                 break;  
             case 1:  
                 type = StartType.LITTLE;  
                 break;  
             case 2:  
                 type = StartType.BIG;  
                 break;  
             default:  
                 break;  
         }  
         leaf.type = type;  
         // 随机起始的旋转角度  
         leaf.rotateAngle = random.nextInt(360);  
         // 随机旋转方向（顺时针或逆时针）  
         leaf.rotateDirection = random.nextInt(2);  
         // 为了产生交错的感觉，让开始的时间有一定的随机性  
         mAddTime += random.nextInt((int) (LEAF_FLOAT_TIME * 1.5));  
         leaf.startTime = System.currentTimeMillis() + mAddTime;  
         return leaf;  
     }  
   
     // 根据最大叶子数产生叶子信息  
     public List<Leaf> generateLeafs() {  
         return generateLeafs(MAX_LEAFS);  
     }  
   
     // 根据传入的叶子数量产生叶子信息  
     public List<Leaf> generateLeafs(int leafSize) {  
         List<Leaf> leafs = new LinkedList<Leaf>();  
         for (int i = 0; i < leafSize; i++) {  
            leafs.add(generateLeaf());  
        }  
         return leafs;  
     }  
 }  

~~~



定义两个常亮分别记录中等振幅和之间的振幅差：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. // 中等振幅大小  
2. private static final int MIDDLE_AMPLITUDE = 13;  
3. // 不同类型之间的振幅差距  
4. private static final int AMPLITUDE_DISPARITY = 5;  

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. // 中等振幅大小  
2. private int mMiddleAmplitude = MIDDLE_AMPLITUDE;  
3. // 振幅差  
4. private int mAmplitudeDisparity = AMPLITUDE_DISPARITY;  

有了以上信息，我们则可以获取到叶子的Y值：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. // 通过叶子信息获取当前叶子的Y值  
2. private int getLocationY(Leaf leaf) {  
3.     // y = A(wx+Q)+h  
4.     float w = (float) ((float) 2 * Math.PI / mProgressWidth);  
5.     float a = mMiddleAmplitude;  
6.     switch (leaf.type) {  
7.         case LITTLE:  
8.             // 小振幅 ＝ 中等振幅 － 振幅差  
9.             a = mMiddleAmplitude - mAmplitudeDisparity;  
10.             break;  
11.         case MIDDLE:  
12.             a = mMiddleAmplitude;  
13.             break;  
14.         case BIG:  
15.             // 小振幅 ＝ 中等振幅 + 振幅差  
16.             a = mMiddleAmplitude + mAmplitudeDisparity;  
17.             break;  
18.         default:  
19.             break;  
20.     }  
21.     Log.i(TAG, "---a = " + a + "---w = " + w + "--leaf.x = " + leaf.x);  
22.     return (int) (a * Math.sin(w * leaf.x)) + mArcRadius * 2 / 3;  
23. }  

接下来，我们开始绘制叶子：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. /**  
2.  * 绘制叶子  
3.  *   
4.  * @param canvas  
5.  */  
6. private void drawLeafs(Canvas canvas) {  
7.     long currentTime = System.currentTimeMillis();  
8.     for (int i = 0; i < mLeafInfos.size(); i++) {  
9.         Leaf leaf = mLeafInfos.get(i);  
10.         if (currentTime > leaf.startTime && leaf.startTime != 0) {  
11.             // 绘制叶子－－根据叶子的类型和当前时间得出叶子的（x，y）  
12.             getLeafLocation(leaf, currentTime);  
13.             // 根据时间计算旋转角度  
14.             canvas.save();  
15.             // 通过Matrix控制叶子旋转  
16.             Matrix matrix = new Matrix();  
17.             float transX = mLeftMargin + leaf.x;  
18.             float transY = mLeftMargin + leaf.y;  
19.             Log.i(TAG, "left.x = " + leaf.x + "--leaf.y=" + leaf.y);  
20.             matrix.postTranslate(transX, transY);  
21.             // 通过时间关联旋转角度，则可以直接通过修改LEAF_ROTATE_TIME调节叶子旋转快慢  
22.             float rotateFraction = ((currentTime - leaf.startTime) % LEAF_ROTATE_TIME)  
23.                     / (float) LEAF_ROTATE_TIME;  
24.             int angle = (int) (rotateFraction * 360);  
25.             // 根据叶子旋转方向确定叶子旋转角度  
26.             int rotate = leaf.rotateDirection == 0 ? angle + leaf.rotateAngle : -angle  
27.                     + leaf.rotateAngle;  
28.             matrix.postRotate(rotate, transX  
29.                     + mLeafWidth / 2, transY + mLeafHeight / 2);  
30.             canvas.drawBitmap(mLeafBitmap, matrix, mBitmapPaint);  
31.             canvas.restore();  
32.         } else {  
33.             continue;  
34.         }  
35.     }  
36. }  

最后，向外层暴露几个接口：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. /**  
2.  * 设置中等振幅  
3.  *   
4.  * @param amplitude  
5.  */  
6. public void setMiddleAmplitude(int amplitude) {  
7.     this.mMiddleAmplitude = amplitude;  
8. }  
9.   
10. /**  
11.  * 设置振幅差  
12.  *   
13.  * @param disparity  
14.  */  
15. public void setMplitudeDisparity(int disparity) {  
16.     this.mAmplitudeDisparity = disparity;  
17. }  
18.   
19. /**  
20.  * 获取中等振幅  
21.  *   
22.  * @param amplitude  
23.  */  
24. public int getMiddleAmplitude() {  
25.     return mMiddleAmplitude;  
26. }  
27.   
28. /**  
29.  * 获取振幅差  
30.  *   
31.  * @param disparity  
32.  */  
33. public int getMplitudeDisparity() {  
34.     return mAmplitudeDisparity;  
35. }  
36.   
37. /**  
38.  * 设置进度  
39.  *   
40.  * @param progress  
41.  */  
42. public void setProgress(int progress) {  
43.     this.mProgress = progress;  
44.     postInvalidate();  
45. }  
46.   
47. /**  
48.  * 设置叶子飘完一个周期所花的时间  
49.  *   
50.  * @param time  
51.  */  
52. public void setLeafFloatTime(long time) {  
53.     this.mLeafFloatTime = time;  
54. }  
55.   
56. /**  
57.  * 设置叶子旋转一周所花的时间  
58.  *   
59.  * @param time  
60.  */  
61. public void setLeafRotateTime(long time) {  
62.     this.mLeafRotateTime = time;  

这些接口用来干嘛呢？用于把我们的动效做成完全可手动调节的，这样做有什么好处呢？

\1. 更加便于产品、射鸡湿查看效果，避免YY，自己手动调节，不会出现要你一遍遍的改参数安装、查看、再改、再查看... ... N遍之后说 “这好像不是我想要的” -- 瞬间天崩地裂，天昏地暗，感觉被全世界抛弃；

\2. 便于体现你是一个考虑全面，思维缜密，会编程、会设计的艺术家，当然这纯属YY，主要还是方便大家；

如此一来，射鸡湿们只需要不断的调节即可实时的看到展现的效果，最后只需要把最终的参数反馈过来即可，万事大吉，一了百了；

当然，如果对方是个漂亮的妹子，而你又苦于没有机会搭讪，以上内容就当我没说，尽情的不按要求写吧，她肯定会主动找你的，说不定连饭都反过来请了... ...

好啦，言归正传，完成收尾部分，我们让所有的参数都可调节起来：

把剩下的layout 和activity贴出来：

activity：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. public class LeafLoadingActivity extends Activity implements OnSeekBarChangeListener,  
2.         OnClickListener {  
3.   
4.     Handler mHandler = new Handler() {  
5.         public void handleMessage(Message msg) {  
6.             switch (msg.what) {  
7.                 case REFRESH_PROGRESS:  
8.                     if (mProgress < 40) {  
9.                         mProgress += 1;  
10.                         // 随机800ms以内刷新一次  
11.                         mHandler.sendEmptyMessageDelayed(REFRESH_PROGRESS,  
12.                                 new Random().nextInt(800));  
13.                         mLeafLoadingView.setProgress(mProgress);  
14.                     } else {  
15.                         mProgress += 1;  
16.                         // 随机1200ms以内刷新一次  
17.                         mHandler.sendEmptyMessageDelayed(REFRESH_PROGRESS,  
18.                                 new Random().nextInt(1200));  
19.                         mLeafLoadingView.setProgress(mProgress);  
20.   
21.                     }  
22.                     break;  
23.   
24.                 default:  
25.                     break;  
26.             }  
27.         };  
28.     };  
29.   
30.     private static final int REFRESH_PROGRESS = 0x10;  
31.     private LeafLoadingView mLeafLoadingView;  
32.     private SeekBar mAmpireSeekBar;  
33.     private SeekBar mDistanceSeekBar;  
34.     private TextView mMplitudeText;  
35.     private TextView mDisparityText;  
36.     private View mFanView;  
37.     private Button mClearButton;  
38.     private int mProgress = 0;  
39.   
40.     private TextView mProgressText;  
41.     private View mAddProgress;  
42.     private SeekBar mFloatTimeSeekBar;  
43.   
44.     private SeekBar mRotateTimeSeekBar;  
45.     private TextView mFloatTimeText;  
46.     private TextView mRotateTimeText;  
47.   
48.     @Override  
49.     protected void onCreate(Bundle savedInstanceState) {  
50.         super.onCreate(savedInstanceState);  
51.         setContentView(R.layout.leaf_loading_layout);  
52.         initViews();  
53.         mHandler.sendEmptyMessageDelayed(REFRESH_PROGRESS, 3000);  
54.     }  
55.   
56.     private void initViews() {  
57.         mFanView = findViewById(R.id.fan_pic);  
58.         RotateAnimation rotateAnimation = DXAnimationUtils.initRotateAnimation(false, 1500, true,  
59.                 Animation.INFINITE);  
60.         mFanView.startAnimation(rotateAnimation);  
61.         mClearButton = (Button) findViewById(R.id.clear_progress);  
62.         mClearButton.setOnClickListener(this);  
63.   
64.         mLeafLoadingView = (LeafLoadingView) findViewById(R.id.leaf_loading);  
65.         mMplitudeText = (TextView) findViewById(R.id.text_ampair);  
66.         mMplitudeText.setText(getString(R.string.current_mplitude,  
67.                 mLeafLoadingView.getMiddleAmplitude()));  
68.   
69.         mDisparityText = (TextView) findViewById(R.id.text_disparity);  
70.         mDisparityText.setText(getString(R.string.current_Disparity,  
71.                 mLeafLoadingView.getMplitudeDisparity()));  
72.   
73.         mAmpireSeekBar = (SeekBar) findViewById(R.id.seekBar_ampair);  
74.         mAmpireSeekBar.setOnSeekBarChangeListener(this);  
75.         mAmpireSeekBar.setProgress(mLeafLoadingView.getMiddleAmplitude());  
76.         mAmpireSeekBar.setMax(50);  
77.   
78.         mDistanceSeekBar = (SeekBar) findViewById(R.id.seekBar_distance);  
79.         mDistanceSeekBar.setOnSeekBarChangeListener(this);  
80.         mDistanceSeekBar.setProgress(mLeafLoadingView.getMplitudeDisparity());  
81.         mDistanceSeekBar.setMax(20);  
82.   
83.         mAddProgress = findViewById(R.id.add_progress);  
84.         mAddProgress.setOnClickListener(this);  
85.         mProgressText = (TextView) findViewById(R.id.text_progress);  
86.   
87.         mFloatTimeText = (TextView) findViewById(R.id.text_float_time);  
88.         mFloatTimeSeekBar = (SeekBar) findViewById(R.id.seekBar_float_time);  
89.         mFloatTimeSeekBar.setOnSeekBarChangeListener(this);  
90.         mFloatTimeSeekBar.setMax(5000);  
91.         mFloatTimeSeekBar.setProgress((int) mLeafLoadingView.getLeafFloatTime());  
92.         mFloatTimeText.setText(getResources().getString(R.string.current_float_time,  
93.                 mLeafLoadingView.getLeafFloatTime()));  
94.   
95.         mRotateTimeText = (TextView) findViewById(R.id.text_rotate_time);  
96.         mRotateTimeSeekBar = (SeekBar) findViewById(R.id.seekBar_rotate_time);  
97.         mRotateTimeSeekBar.setOnSeekBarChangeListener(this);  
98.         mRotateTimeSeekBar.setMax(5000);  
99.         mRotateTimeSeekBar.setProgress((int) mLeafLoadingView.getLeafRotateTime());  
100.         mRotateTimeText.setText(getResources().getString(R.string.current_float_time,  
101.                 mLeafLoadingView.getLeafRotateTime()));  
102.     }  
103.   
104.     @Override  
105.     public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {  
106.         if (seekBar == mAmpireSeekBar) {  
107.             mLeafLoadingView.setMiddleAmplitude(progress);  
108.             mMplitudeText.setText(getString(R.string.current_mplitude,  
109.                     progress));  
110.         } else if (seekBar == mDistanceSeekBar) {  
111.             mLeafLoadingView.setMplitudeDisparity(progress);  
112.             mDisparityText.setText(getString(R.string.current_Disparity,  
113.                     progress));  
114.         } else if (seekBar == mFloatTimeSeekBar) {  
115.             mLeafLoadingView.setLeafFloatTime(progress);  
116.             mFloatTimeText.setText(getResources().getString(R.string.current_float_time,  
117.                     progress));  
118.         }  
119.         else if (seekBar == mRotateTimeSeekBar) {  
120.             mLeafLoadingView.setLeafRotateTime(progress);  
121.             mRotateTimeText.setText(getResources().getString(R.string.current_rotate_time,  
122.                     progress));  
123.         }  
124.   
125.     }  
126.   
127.     @Override  
128.     public void onStartTrackingTouch(SeekBar seekBar) {  
129.   
130.     }  
131.   
132.     @Override  
133.     public void onStopTrackingTouch(SeekBar seekBar) {  
134.   
135.     }  
136.   
137.     @Override  
138.     public void onClick(View v) {  
139.         if (v == mClearButton) {  
140.             mLeafLoadingView.setProgress(0);  
141.             mHandler.removeCallbacksAndMessages(null);  
142.             mProgress = 0;  
143.         } else if (v == mAddProgress) {  
144.             mProgress++;  
145.             mLeafLoadingView.setProgress(mProgress);  
146.             mProgressText.setText(String.valueOf(mProgress));  
147.         }  
148.     }  
149. }  

layout：

**[html]** [view plain](https://blog.csdn.net/tianjian4592/article/details/44538605#) [copy](https://blog.csdn.net/tianjian4592/article/details/44538605#)

1. <?xml version="1.0" encoding="utf-8"?>  
2. <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
3.     android:layout_width="match_parent"  
4.     android:layout_height="match_parent"  
5.     android:background="#fed255"  
6.     android:orientation="vertical" >  
7.   
8.     <TextView  
9.         android:layout_width="wrap_content"  
10.         android:layout_height="wrap_content"  
11.         android:layout_gravity="center_horizontal"  
12.         android:layout_marginTop="100dp"  
13.         android:text="loading ..."  
14.         android:textColor="#FFA800"  
15.         android:textSize=" 30dp" />  
16.   
17.     <RelativeLayout  
18.         android:id="@+id/leaf_content"  
19.         android:layout_width="match_parent"  
20.         android:layout_height="wrap_content"  
21.         android:layout_marginTop="50dp" >  
22.   
23.         <com.baidu.batterysaverDemo.ui.LeafLoadingView  
24.             android:id="@+id/leaf_loading"  
25.             android:layout_width="302dp"  
26.             android:layout_height="61dp"  
27.             android:layout_centerHorizontal="true" />  
28.   
29.         <ImageView  
30.             android:id="@+id/fan_pic"  
31.             android:layout_width="wrap_content"  
32.             android:layout_height="wrap_content"  
33.             android:layout_alignParentRight="true"  
34.             android:layout_centerVertical="true"  
35.             android:layout_marginRight="35dp"  
36.             android:src="@drawable/fengshan" />  
37.     </RelativeLayout>  
38.   
39.     <ScrollView  
40.         android:layout_width="match_parent"  
41.         android:layout_height="match_parent" >  
42.   
43.         <LinearLayout  
44.             android:layout_width="match_parent"  
45.             android:layout_height="match_parent"  
46.             android:orientation="vertical" >  
47.   
48.             <LinearLayout  
49.                 android:id="@+id/seek_content_one"  
50.                 android:layout_width="match_parent"  
51.                 android:layout_height="wrap_content"  
52.                 android:layout_marginLeft="15dp"  
53.                 android:layout_marginRight="15dp"  
54.                 android:layout_marginTop="15dp" >  
55.   
56.                 <TextView  
57.                     android:id="@+id/text_ampair"  
58.                     android:layout_width="wrap_content"  
59.                     android:layout_height="wrap_content"  
60.                     android:layout_gravity="center_vertical"  
61.                     android:textColor="#ffffa800"  
62.                     android:textSize="15dp" />  
63.   
64.                 <SeekBar  
65.                     android:id="@+id/seekBar_ampair"  
66.                     android:layout_width="0dp"  
67.                     android:layout_height="wrap_content"  
68.                     android:layout_marginLeft="5dp"  
69.                     android:layout_weight="1" />  
70.             </LinearLayout>  
71.   
72.             <LinearLayout  
73.                 android:layout_width="match_parent"  
74.                 android:layout_height="wrap_content"  
75.                 android:layout_marginLeft="15dp"  
76.                 android:layout_marginRight="15dp"  
77.                 android:layout_marginTop="15dp"  
78.                 android:orientation="horizontal" >  
79.   
80.                 <TextView  
81.                     android:id="@+id/text_disparity"  
82.                     android:layout_width="wrap_content"  
83.                     android:layout_height="wrap_content"  
84.                     android:layout_gravity="center_vertical"  
85.                     android:textColor="#ffffa800"  
86.                     android:textSize="15dp" />  
87.   
88.                 <SeekBar  
89.                     android:id="@+id/seekBar_distance"  
90.                     android:layout_width="0dp"  
91.                     android:layout_height="wrap_content"  
92.                     android:layout_marginLeft="5dp"  
93.                     android:layout_weight="1" />  
94.             </LinearLayout>  
95.   
96.             <LinearLayout  
97.                 android:layout_width="match_parent"  
98.                 android:layout_height="wrap_content"  
99.                 android:layout_marginLeft="15dp"  
100.                 android:layout_marginRight="15dp"  
101.                 android:layout_marginTop="15dp"  
102.                 android:orientation="horizontal" >  
103.   
104.                 <TextView  
105.                     android:id="@+id/text_float_time"  
106.                     android:layout_width="wrap_content"  
107.                     android:layout_height="wrap_content"  
108.                     android:layout_gravity="center_vertical"  
109.                     android:textColor="#ffffa800"  
110.                     android:textSize="15dp" />  
111.   
112.                 <SeekBar  
113.                     android:id="@+id/seekBar_float_time"  
114.                     android:layout_width="0dp"  
115.                     android:layout_height="wrap_content"  
116.                     android:layout_marginLeft="5dp"  
117.                     android:layout_weight="1" />  
118.             </LinearLayout>  
119.   
120.             <LinearLayout  
121.                 android:layout_width="match_parent"  
122.                 android:layout_height="wrap_content"  
123.                 android:layout_marginLeft="15dp"  
124.                 android:layout_marginRight="15dp"  
125.                 android:layout_marginTop="15dp"  
126.                 android:orientation="horizontal" >  
127.   
128.                 <TextView  
129.                     android:id="@+id/text_rotate_time"  
130.                     android:layout_width="wrap_content"  
131.                     android:layout_height="wrap_content"  
132.                     android:layout_gravity="center_vertical"  
133.                     android:textColor="#ffffa800"  
134.                     android:textSize="15dp" />  
135.   
136.                 <SeekBar  
137.                     android:id="@+id/seekBar_rotate_time"  
138.                     android:layout_width="0dp"  
139.                     android:layout_height="wrap_content"  
140.                     android:layout_marginLeft="5dp"  
141.                     android:layout_weight="1" />  
142.             </LinearLayout>  
143.   
144.             <Button  
145.                 android:id="@+id/clear_progress"  
146.                 android:layout_width="match_parent"  
147.                 android:layout_height="wrap_content"  
148.                 android:layout_marginTop="15dp"  
149.                 android:text="去除进度条,玩转弧线"  
150.                 android:textSize="18dp" />  
151.   
152.             <LinearLayout  
153.                 android:layout_width="match_parent"  
154.                 android:layout_height="wrap_content"  
155.                 android:layout_marginLeft="15dp"  
156.                 android:layout_marginRight="15dp"  
157.                 android:layout_marginTop="15dp"  
158.                 android:orientation="horizontal" >  
159.   
160.                 <Button  
161.                     android:id="@+id/add_progress"  
162.                     android:layout_width="wrap_content"  
163.                     android:layout_height="wrap_content"  
164.                     android:text="增加进度: "  
165.                     android:textSize="18dp" />  
166.   
167.                 <TextView  
168.                     android:id="@+id/text_progress"  
169.                     android:layout_width="wrap_content"  
170.                     android:layout_height="wrap_content"  
171.                     android:layout_gravity="center_vertical"  
172.                     android:textColor="#ffffa800"  
173.                     android:textSize="15dp" />  
174.             </LinearLayout>  
175.         </LinearLayout>  
176.     </ScrollView>  
177.   
178. </LinearLayout>  

最终效果如下，本来录了20+s，但是PS只能转5s，所以有兴趣的大家自己运行的玩吧：

![](https://img-blog.csdn.net/20150323014910130?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdGlhbmppYW40NTky/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

github 源码地址：[https://github.com/Ajian-studio/GALeafLoading](https://github.com/Ajian-studio/GALeafLoading)

注：后续 github 会开始新项目更新，大家可以保持关注；

