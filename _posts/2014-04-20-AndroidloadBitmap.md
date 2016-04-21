---
layout: default 
title: Android 高效处理图片 

---
本文是我参考Android的官方文档写的，官网[地址](https://developer.android.com/training/displaying-bitmaps/index.html)    

# 高效加载图片  

![我的头像]({{ site.baseurl }}/assets/head.jpg '可选的标题')  
一般来说现在的手机拍照的图片动辄几兆甚至十几兆，对于Android手机来说默认的单个应用的可分配的内存只有16M，所以说如何处理图片是  
android 上一个不小的难题。
根据不同的图片来源（有网络的图片、本地图片等）Android官方提供了BitmapFactory 这个工具类，可以使用decodeByteArray(),   decodeFile(), decodeResource()等用来生成Bitmap，但是这里面如果图片的资源过大非常容易导致OOM。下面的写法是比较通用的避免类似的问题。  
{% highlight java linenos %}
BitmapFactory.Options options = new BitmapFactory.Options();  
options.inJustDecodeBounds = true;  
BitmapFactory.decodeResource(getResources(), R.id.myimage, options);  
int imageHeight = options.outHeight;  
int imageWidth = options.outWidth;  
String imageType = options.outMimeType;  
{% endhighlight %}
这里面主要通过设置 options.inJustDecodeBounds = true;让我们不但可以读出资源图片的type,宽和高，而且最重要的是不会产生实际的bitmap,换句话说就是不会占用内存，也就不会出现OOM的情况。
拿到图片的宽高，再去做处理就比较简单了，不过也要根据不同的需求做不同的处理（主要根据需求：例如显示在多大的imagview上面，屏幕的尺  寸和分辨率。。。）。
使用inSampleSize 对图片进行等比压缩。例如：inSampleSize=4，就是把图片压缩为原来的1/4。
{% highlight java linenos %}
public static int calculateInSampleSize(
    BitmapFactory.Options options, int reqWidth, int reqHeight) {
// Raw height and width of image
final int height = options.outHeight;
final int width = options.outWidth;
int inSampleSize = 1;

if (height > reqHeight || width > reqWidth) {

final int halfHeight = height / 2;
final int halfWidth = width / 2;

// Calculate the largest inSampleSize value that is a power of 2 and keeps both
// height and width larger than the requested height and width.
while ((halfHeight / inSampleSize) > reqHeight
        && (halfWidth / inSampleSize) > reqWidth) {
    inSampleSize *= 2;
}
}

return inSampleSize;
}
	{% endhighlight %}

{% highlight java  linenos %}
public static Bitmap decodeSampledBitmapFromResource(Resources res, int resId,
    int reqWidth, int reqHeight) {

// First decode with inJustDecodeBounds=true to check dimensions
final BitmapFactory.Options options = new BitmapFactory.Options();
options.inJustDecodeBounds = true;
BitmapFactory.decodeResource(res, resId, options);

// Calculate inSampleSize
options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

// Decode bitmap with inSampleSize set
options.inJustDecodeBounds = false;
return BitmapFactory.decodeResource(res, resId, options);
}
mImageView.setImageBitmap(
decodeSampledBitmapFromResource(getResources(), R.id.myimage, 100, 100));
{% endhighlight %}
这里面演示的是使用本地的资源图片，真正使用过程中会加载各种资源，根据资源的不同使用BitmapFactory.decode的不  
同的方法就可以了。

# 子线程中处理bitmaps
	 
## 使用Asynctask

* BitmapFactory.decode方法不应该在主线程中调用，因为这会阻塞主线程导致用户体验不好。  

{% highlight java linenos %}
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    private final WeakReference<ImageView> imageViewReference;
    private int data = 0;

    public BitmapWorkerTask(ImageView imageView) {
        // Use a WeakReference to ensure the ImageView can be garbage collected
        imageViewReference = new WeakReference<ImageView>(imageView);
    }

    // Decode image in background.
    @Override
    protected Bitmap doInBackground(Integer... params) {
        data = params[0];
        return decodeSampledBitmapFromResource(getResources(), data, 100, 100));
    }

    // Once complete, see if ImageView is still around and set bitmap.
    @Override
    protected void onPostExecute(Bitmap bitmap) {
        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            if (imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
}
{% endhighlight %}

*  使用WeakReference，主要是确保在必要的时候可以被垃圾回收器回收，在onPostExecute对图片进行操作  
	前需要进行检查，因为不能保证imageView一定存在，因为用户可能已经离开这个界面了，或者其他因素等等。。。  

*  开始任务的代码就很简单了

{% highlight java linenos %}
public void loadBitmap(int resId, ImageView imageView) {
    BitmapWorkerTask task = new BitmapWorkerTask(imageView);
    task.execute(resId);
}
{% endhighlight %}

##处理并发

* 一般的控件像ListView和Gridview结合Asynctask加载图片的时候会有新的问题，为了提高使用效率，  

当用户滑动的时候这些控件会重复利用子控件，如果每一个imageview都触发一个Task，当task结束的时候，对应的  
imageview可能已经被复用了，或者被别的控件使用了，更重要的是我们不能保证task结束的顺序。这篇[博客](http://android-developers.blogspot.com/2010/07/multithreading-for-performance.html)深入讨论了并发的问题，并且提出了  
不错的解决方案，接下来我们就模仿它写个类似的方案。

* 自定义一个继承BitmapDrawable的专门的类，它的作用就是在Task结束的时候用来展示imageview的内容。

{% highlight java linenos %}
static class AsyncDrawable extends BitmapDrawable {
    private final WeakReference<BitmapWorkerTask> bitmapWorkerTaskReference;

    public AsyncDrawable(Resources res, Bitmap bitmap,
            BitmapWorkerTask bitmapWorkerTask) {
        super(res, bitmap);
        bitmapWorkerTaskReference =
            new WeakReference<BitmapWorkerTask>(bitmapWorkerTask);
    }

    public BitmapWorkerTask getBitmapWorkerTask() {
        return bitmapWorkerTaskReference.get();
    }
}
{% endhighlight %}

* 在执行BitmapWorkerTask之前，你可以创建一个AsyncDrawable，并且绑定Imagview。

* 下面的方法是为了检测imageview是否有相关的Task启动了

{% highlight java linenos %}
private static BitmapWorkerTask getBitmapWorkerTask(ImageView imageView) {
   if (imageView != null) {
       final Drawable drawable = imageView.getDrawable();
       if (drawable instanceof AsyncDrawable) {
           final AsyncDrawable asyncDrawable = (AsyncDrawable) drawable;
           return asyncDrawable.getBitmapWorkerTask();
       }
    }
    return null;
}
{% endhighlight %}

* 是否取消之前的任务（这里比较复杂：详见代码分析）  

{% highlight java linenos %}
public static boolean cancelPotentialWork(int data, ImageView imageView) {
    final BitmapWorkerTask bitmapWorkerTask = getBitmapWorkerTask(imageView);

    if (bitmapWorkerTask != null) {
        final int bitmapData = bitmapWorkerTask.data;
        // If bitmapData is not yet set or it differs from the new data
        if (bitmapData == 0 || bitmapData != data) {
            // Cancel previous task
            bitmapWorkerTask.cancel(true);
        } else {
            // The same work is already in progress
            return false;
        }
    }
    // No task associated with the ImageView, or an existing task was cancelled
    return true;
}
{% endhighlight %}

* 加载图片

{% highlight java linenos %}
public void loadBitmap(int resId, ImageView imageView) {
    if (cancelPotentialWork(resId, imageView)) {
        final BitmapWorkerTask task = new BitmapWorkerTask(imageView);
        final AsyncDrawable asyncDrawable =
                new AsyncDrawable(getResources(), mPlaceHolderBitmap, task);
        imageView.setImageDrawable(asyncDrawable);
        task.execute(resId);
    }
}
{% endhighlight %}  

* 最后一步就是在onpost中设置图片的内容

{% highlight java linenos %}
@Override
    protected void onPostExecute(Bitmap bitmap) {
        if (isCancelled()) {
            bitmap = null;
        }

        if (imageViewReference != null && bitmap != null) {
            final ImageView imageView = imageViewReference.get();
            final BitmapWorkerTask bitmapWorkerTask =
                    getBitmapWorkerTask(imageView);
            if (this == bitmapWorkerTask && imageView != null) {
                imageView.setImageBitmap(bitmap);
            }
        }
    }
{% endhighlight %}





