Android RecyclerView: Super Fast ListView
===

先上图，看效果：

![img](https://dl.dropboxusercontent.com/u/96034496/moments/super-fast-momoso.jpg)

前几天刚release完公司的一个项目，有了点时间，于是就想找一些有意思的东西学习一下，顺便运用在项目之中。看到iOS的同事们在谈论iOS8的xx特性时，我突然也有想在公司项目的下一个版本中添加Android L版本的特性。

六月底的时候收看Google io时，当时对Android新的设计语言，Material Design，没什么太大的好感，感觉色彩一坨一坨的，好难看的样子，当时觉得亮点就是新的ART运行时环境和一些酷炫的动画效果。再后来，8月初的时候，自己出于好奇真的拿Nexus 5安装了一个L的预览版，体验很差...好多软件都还是holo的，反正觉得不是很期待就是啦。

回到重点，下载好最新的SDK，你会发现在``ANDROID_HOME/extras/android/m2repository/com/android/support``下面多了不少兼容库，**cardview**, **support-annotations**, **recyclerview-v7**，眼前一亮吧~这回，Google真的是拿出了好多东西呀，赞，尤其是cardview和recyclerview这两个新的控件，这个在Google最新的Material Design主页上有说明和简单的介绍，简而言之，cardview可以提供和Google很多自家应用观感一致的卡片化布局，而recyclerview则是一个增强版的listview，更强大和好用。

手痒了，特别想试试，但是这里有一个坑，因为仍旧是预览版，所以Google把minSDKVersion设置成了L，意思就是只有使用L预览版系统的机器才可以测试。呵呵，广大人民群众怎么会被这个给吓到，网上有在AndroidManifest.xml中设置``<uses-sdk tools:node="replace" />``即可。还有另一招，将源码解压出来，然后自己按照项目结构放置源码文件，最后再在自己的项目中引入就好了，但是要注意一点，需要把L版本相关的代码给删掉，无所谓啦，反正到时候Google推出正式版的。

废话扯了那么多，下面才是今天的主题，super fast listview，从来没有见过这样快的list，甚至还支持横向的滚动，要知道，这在之前的Android，要实现横向的list是有多蛋疼！还有更多的惊喜，在另一个兼容库``leanback-v17``中，还有Grid，StagedGrid，HorizonalGrid等更高级的Widget，知道Pinterest的瀑布流麽？

下面的代码，提供了滑动到底部自动加载更多的功能，是我自己根据以前listview的经验写的，由于加载的速度过快，在删除加载更多的提示时，有时会出现页面有一部分空白间距的问题，没办法，只好postdelay 50毫秒，再将加载回来的list追加到末尾。

下面是源代码，使用recycle view配合card view实现无限list（自动带提示加载更多，并且包含不用类型的view），super fast~ 看这段代码前希望你能先去Material Design的主页看看基本介绍和范例代码。

``MainActivity.java``

```java
/*
 * Copyright (c) 2014 longkai
 * The software shall be used for good, not evil.
 */

package com.example.gridlayout;

import android.app.Activity;
import android.app.Fragment;
import android.os.Bundle;
import android.os.Handler;
import android.support.annotation.Nullable;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.util.Log;
import android.view.LayoutInflater;
import android.view.Menu;
import android.view.MenuItem;
import android.view.View;
import android.view.ViewGroup;
import android.widget.TextView;
import com.manuelpeinado.refreshactionitem.RefreshActionItem;
import org.json.JSONException;
import org.json.JSONObject;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class MainActivity extends Activity {
  public static final String TAG = MainActivity.class.getSimpleName();

  public static final String TYPE = "type";
  public static final int ITEM = 0;
  public static final int SIMPLE = 1;
  public static final int FOOTER = 2;

  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    if (savedInstanceState == null) {
      try {
        getFragmentManager().beginTransaction()
            .replace(android.R.id.content, CardFragment.class.newInstance())
            .commit();
      } catch (InstantiationException e) {
        e.printStackTrace();
      } catch (IllegalAccessException e) {
        e.printStackTrace();
      }
    }
  }

  @Override public boolean onCreateOptionsMenu(Menu menu) {
    getMenuInflater().inflate(R.menu.main, menu);
    return true;
  }

  @Override public boolean onMenuItemSelected(int featureId, MenuItem item) {
    switch (item.getItemId()) {
      case R.id.action_settings:
        break;
      default:
        break;
    }
    return super.onMenuItemSelected(featureId, item);
  }

  public static class CardFragment extends Fragment {

    boolean loading = false;

    Handler mHandler = new Handler();
    RecyclerView mRecyclerView;
    CardAdapter mAdapter;

    @Nullable @Override public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
      View view = inflater.inflate(R.layout.recycler, container, false);
      mRecyclerView = (RecyclerView) view.findViewById(R.id.recycler);
      return view;
    }

    @Override public void onViewCreated(View view, Bundle savedInstanceState) {
      super.onViewCreated(view, savedInstanceState);
    }

    @Override public void onActivityCreated(Bundle savedInstanceState) {
      super.onActivityCreated(savedInstanceState);
      List<JSONObject> list = new ArrayList<>();
      try {
        for (int i = 0; i < 300; i++) {
          JSONObject jsonObject = new JSONObject();
          if (i % 10 == 0) {
            jsonObject.put(TYPE, SIMPLE);
          } else {
            jsonObject.put(TYPE, ITEM);
          }
          list.add(jsonObject);
        }
      } catch (JSONException ignore) {
      }
      mAdapter = new CardAdapter(list);
      mRecyclerView.setHasFixedSize(true);
      mRecyclerView.setAdapter(mAdapter);
      LinearLayoutManager layoutManager = new LinearLayoutManager(getActivity());
      layoutManager.setOrientation(LinearLayoutManager.VERTICAL);
      mRecyclerView.setLayoutManager(layoutManager);
      mRecyclerView.setOnScrollListener(new RecyclerView.OnScrollListener() {
        @Override public void onScrollStateChanged(int newState) {

        }

        @Override public void onScrolled(int dx, int dy) {
          if (!loading && layoutManager.findLastVisibleItemPosition() == list.size() - 1) {
            loading = true;
            JSONObject jsonObject = new JSONObject();
            try {
              jsonObject.put(TYPE, FOOTER);
            } catch (JSONException ignore) {
            }
            mAdapter.add(jsonObject);
            new Thread(() -> {
              try {
                TimeUnit.SECONDS.sleep(3);
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
              List<JSONObject> tmp = new ArrayList<JSONObject>();
              for (int i = 0; i < 10; i++) {
                JSONObject json = new JSONObject();
                try {
                  json.put(TYPE, ITEM);
                } catch (JSONException e) {
                }
                tmp.add(json);
              }
              mHandler.post(() -> {
                mAdapter.remove(mAdapter.getItemCount() - 1);
                mAdapter.addAll(tmp);
                loading = false;
              });
            }).start();
          }
        }
      });
    }
  }

  private static class CardAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    private List<JSONObject> list;

    private CardAdapter(List<JSONObject> list) {
      this.list = list;
    }

    @Override public int getItemCount() {
      return list.size();
    }

    @Override public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
      switch (viewType) {
        case SIMPLE:
          return new RecyclerView.ViewHolder(LayoutInflater.from(parent.getContext()).inflate(android.R.layout.simple_list_item_1, parent, false)) {
          };
        case FOOTER:
          return new RecyclerView.ViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.load_more, parent, false)) {
          };
        default:
        case ITEM:
          View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.card, parent, false);
          CardViewHolder holder = new CardViewHolder(view);
          return holder;
      }
    }

    @Override public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
      switch (list.get(position).optInt(TYPE)) {
        case ITEM:
          CardViewHolder cardViewHolder = (CardViewHolder) holder;
          cardViewHolder.textView.setText("item " + position);
          break;
        case SIMPLE:
          TextView txt = (TextView) holder.itemView.findViewById(android.R.id.text1);
          txt.setText("simple txt!");
          Log.d(TAG, "simple text");
          break;
        case FOOTER:
          Log.d(TAG, "footer!");
          break;
      }
    }

    @Override public int getItemViewType(int position) {
      return list.get(position).optInt(TYPE);
    }

    public void add(JSONObject jsonObject) {
      this.list.add(jsonObject);
      notifyItemInserted(list.size() - 1);
    }

    public void addAll(List<JSONObject> list) {
      this.list.addAll(list);
      notifyDataSetChanged();
    }

    public void remove(int i) {
      list.remove(i);
      notifyItemRemoved(i);
    }

    static class CardViewHolder extends RecyclerView.ViewHolder {
      TextView textView;

      CardViewHolder(View view) {
        super(view);
        textView = (TextView) view.findViewById(R.id.txt);
      }
    }
  }
}
```

以下是布局文件，非常简单

``card.xml``

```xml
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:card_view="http://schemas.android.com/apk/res-auto"
    style="@style/CardView.Light"
    card_view:cardCornerRadius="4dp"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

  <TextView
      android:id="@+id/txt"
      android:textAppearance="?android:textAppearanceMedium"
      android:gravity="center"
      android:layout_width="match_parent"
      android:layout_height="200dp" />
</android.support.v7.widget.CardView>
```

``recycler.xml``

```xml
<android.support.v7.widget.RecyclerView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/recycler"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

最后附一张效果图，比较丑，只是为了演示而已

![simple](https://dl.dropboxusercontent.com/u/96034496/moments/simple-recycler-card.jpg)

由于是公司的项目，所以比较详细的代码没有贴出来，但是也是依据这段代码弄出来的，有时间的话，改天封装一个出来~

---
by longkai on 1 Sep. in Sz.


### EOF
```yaml
background: ""
date: 2014-09-01T22:16:22+08:00
hide: false
location: Shenzhen
summary: ""
tags:
- Android
weather: ""
```
