
#### 效果
<p>
<img src="imgs/device-2022-08-04-160814 00_00_00-00_00_30.gif" width="40%">
</p>

 
 ## 简介
 
 项目需要做一个拖拽排序的需求(类似头条栏目排序),原先随意找了个三方库简单的处理了一下.但是随着项目的的迭代,越来越多的需求堆积下来,三方库不满足自己定制的一些需求.所以决定自己写一写这个效果
 
 ## 思路:
 
 RecycleView 实现列表样式,ItemTouchHelper实现子条目的拖拽和侧滑删除.中间牵扯到指定条目禁止排序,禁止删除的功能.
 
 ## 实现
 
 #### 1.页面搭建
 
 1.1 主页面Activity代码
 ```
 
 package com.wkq.dragrecycle
 
 import androidx.appcompat.app.AppCompatActivity
 import android.os.Bundle
 import android.widget.Toast
 import androidx.databinding.DataBindingUtil
 import androidx.recyclerview.widget.GridLayoutManager
 import androidx.recyclerview.widget.ItemTouchHelper
 import androidx.recyclerview.widget.LinearLayoutManager
 import com.wkq.dragrecycle.databinding.ActivityMainBinding
 
 class MainActivity : AppCompatActivity() {
 
     var stringList = arrayListOf<String>("北京", "河北", "河南", "山东", "天津", "陕西",
         "山西", "石家庄", "内蒙古", "黑龙江", "吉林", "新疆", "西藏", "安徽", "湖北"
     )
 
     private lateinit var mGridItemDecoration: GridSpaceItemDecoration
     var binding: ActivityMainBinding? = null
     override fun onCreate(savedInstanceState: Bundle?) {
         super.onCreate(savedInstanceState)
         binding = DataBindingUtil.setContentView<ActivityMainBinding>(this, R.layout.activity_main)
         initView()
 
     }
 
     private fun initView() {
         //设置布局选择
         binding!!.btChange.setOnClickListener {
             when ( binding!!.rvContent.layoutManager) {
                 is GridLayoutManager -> {
                     binding!!.rvContent.layoutManager = LinearLayoutManager(this)
                 }
                 else -> {
                     binding!!.rvContent.layoutManager = GridLayoutManager(this, 4)
                 }
             }
         }
 
         //默认网格布局
         var mAdapter = DragAdapter(this)
         mGridItemDecoration = GridSpaceItemDecoration(4)
         binding!!.rvContent.addItemDecoration(mGridItemDecoration, 0)
         binding!!.rvContent.layoutManager = GridLayoutManager(this, 4)
         binding!!.rvContent.adapter = mAdapter
         mAdapter.addItems(stringList)
 
         //拖拽绑定的监听器
         var callBack = DragItemCallback(mAdapter,true)
         //拖拽触摸的帮助类
         var itemTouchHelper = ItemTouchHelper(callBack)
         //绑定Rv
         itemTouchHelper.attachToRecyclerView(binding!!.rvContent)
 
         mAdapter.setOnItemClickListener(object : DragAdapter.OnItemClickListener {
             override fun onItemClick(position: Int) {
                 Toast.makeText(this@MainActivity, mAdapter.getItems().get(position), Toast.LENGTH_SHORT).show()
             }
 
             override fun onItemLongClick(holder: DragAdapterViewHolder) {
                 //长按开启拖拽
                 itemTouchHelper.startDrag(holder)
             }
         })
 
     }
 }
 
 ```
 
 1.2  Adapter 代码实现(注意默认禁止操作Item的位置默认0)
 
 ```
 package com.wkq.dragrecycle
 
 import android.content.Context
 import android.view.LayoutInflater
 import android.view.View
 import android.view.ViewGroup
 import androidx.databinding.DataBindingUtil
 import androidx.recyclerview.widget.RecyclerView
 import com.wkq.dragrecycle.databinding.ItemDragBinding
 
 
 /**
  * @author wkq
  *
  * @date 2022年08月04日 13:05
  *
  * @des  拖拽的Adapter
  *  
  * @param limitPosition 禁止操作的条目位置 默认第一个
  *
  */
 
 class DragAdapter(mContext: Context,limitPosition:Int=0) : RecyclerView.Adapter<DragAdapterViewHolder>() {
     
     //上下文
     var mContext=mContext
     // 固定条目的位置
     val limitPosition = limitPosition
     //内容数据
    private var contentList = ArrayList<String>()
     override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): DragAdapterViewHolder {
         var binding = DataBindingUtil.inflate<ItemDragBinding>(
             LayoutInflater.from(mContext),
             R.layout.item_drag,
             parent,
             false
         )
         var holder = DragAdapterViewHolder(binding.root)
         holder.setDataBinding(binding)
         return holder
     }
 
     override fun onBindViewHolder(holder: DragAdapterViewHolder, position: Int) {
         var binding = holder.getBinding() as ItemDragBinding
 
         if (limitPosition==position){
             binding.tvContent.setBackgroundResource(R.drawable.shape_radius5_green2)
         }else{
             binding.tvContent.setBackgroundResource(R.drawable.shape_radius5_green)
         }
         binding.tvContent.text = contentList.get(position)
 
         binding.tvContent.setOnClickListener {
             mListener?.onItemClick(holder.adapterPosition)
         }
         binding.tvContent.setOnLongClickListener {
             mListener?.onItemLongClick(holder)
             return@setOnLongClickListener true
         }
     }
 
     override fun getItemCount(): Int {
         return contentList.size
     }
 
     fun addItems(items: ArrayList<String>) {
         contentList.addAll(items)
     }
 
     fun addItems(item: String) {
         contentList.add(item)
     }
 
     fun remove(item: String) {
         contentList.remove(item)
         notifyDataSetChanged()
     }
 
     fun getItems():ArrayList<String>{
         return contentList
     }
     private var mListener: OnItemClickListener? = null
 
     fun setOnItemClickListener(listener: OnItemClickListener) {
         mListener = listener
     }
 
     interface OnItemClickListener {
         fun onItemClick(position: Int)
         fun onItemLongClick(holder: DragAdapterViewHolder)
     }
 
 }
 
 ```
 
 #### 2. DragItemCallback 拖拽的接口回调(主要实现逻辑)
 
 ```
 package com.wkq.dragrecycle
 
 import android.graphics.drawable.GradientDrawable
 import android.util.Log
 import androidx.core.content.ContextCompat
 import androidx.core.view.ViewCompat
 import androidx.recyclerview.widget.GridLayoutManager
 import androidx.recyclerview.widget.ItemTouchHelper
 import androidx.recyclerview.widget.LinearLayoutManager
 import androidx.recyclerview.widget.RecyclerView
 import java.util.*
 
 
 /**
  * @author wkq
  *
  * @date 2022年08月04日 13:27
  *
  *@des  拖拽的接口回调
  *
  *@param isSwipe  是否支持侧滑  默认支持
  *
  */
 
 class DragItemCallback(adapter: DragAdapter, isSwipe: Boolean = true) : ItemTouchHelper.Callback() {
 
     private var mAdapter = adapter
     private var mData = adapter.getItems()
     private var isSwipe = isSwipe
 
     // 这个方法用于让RecyclerView拦截向上滑动，向下滑动，想左滑动
     // makeMovementFlags(dragFlags, swipeFlags); dragFlags显示项目支持拖动的方向  swipe标记物品可以滑动的方向  0 表示禁止
     //  网格布局:上下左右
     //  线性布局:上下/左右
 
     override fun getMovementFlags(
         recyclerView: RecyclerView,
         viewHolder: RecyclerView.ViewHolder
     ): Int {
         //拖动标记 0 表示禁止
         var dragFlags = 0
         //侧滑标记
         var swipeFlags = 0
         var layoytManager = recyclerView.layoutManager
         if (layoytManager is GridLayoutManager) {
             //网格布局  默认禁止侧滑
             if (viewHolder.adapterPosition != mAdapter.limitPosition) {
                 dragFlags =ItemTouchHelper.LEFT or ItemTouchHelper.UP or ItemTouchHelper.RIGHT or ItemTouchHelper.DOWN
             }
             return makeMovementFlags(dragFlags, swipeFlags)
         } else if (layoytManager is LinearLayoutManager) {
 
             //处理 禁止滑动模块得逻辑  禁止移动得模块 禁止左右滑动
             if (viewHolder.adapterPosition != mAdapter.limitPosition) {
                 dragFlags = ItemTouchHelper.UP or ItemTouchHelper.DOWN
                 swipeFlags = ItemTouchHelper.START or ItemTouchHelper.END
             }
             return makeMovementFlags(dragFlags, swipeFlags)
         }
         //默认 禁止滑动  禁止拖拽
         return makeMovementFlags(dragFlags, swipeFlags)
 
     }
 
     /**
      * 拖拽移动得回调
      */
     override fun onMove(
         recyclerView: RecyclerView,
         viewHolder: RecyclerView.ViewHolder,
         target: RecyclerView.ViewHolder
     ): Boolean {
         // 起始位置
         val fromPosition = viewHolder.adapterPosition
         // 结束位置
         val toPosition = target.adapterPosition
         // 固定位置  处理笃定位置不要移动
         if (fromPosition == mAdapter.limitPosition || toPosition == mAdapter.limitPosition) {
             return false
         }
         // 根据滑动方向 交换数据  for 循环不包含  toPosition  1 替代2     2替代3  3 替代4
         if (fromPosition < toPosition) {
             // 含头不含尾
             for (index in fromPosition until toPosition) {
                 //交换集合得位置
                 Collections.swap(mData, index, index + 1)
             }
         } else {
             // 含头不含尾
             for (index in fromPosition downTo toPosition + 1) {
                 Collections.swap(mData, index, index - 1)
             }
         }
         // 刷新布局
         mAdapter.notifyItemMoved(fromPosition, toPosition)
         return true
     }
 
     /**
      * 滑动结束得回调 滑动删除得逻辑
      */
     override fun onSwiped(viewHolder: RecyclerView.ViewHolder, direction: Int) {
         // direction 滑动的状态
         //ItemTouchHelper.START表示向左滑动
         // ItemTouchHelper.END  向右边滑动
         val position = viewHolder.adapterPosition
         if (position != mAdapter.limitPosition) {
             //表示  禁止移动得布局
             mData.removeAt(position)
             mAdapter.notifyItemRemoved(position)
         }
 
     }
 
 
     /**
      * 选中会回调这里  处理选中布局的展示样式
      */
     override fun onSelectedChanged(viewHolder: RecyclerView.ViewHolder?, actionState: Int) {
 
         if (viewHolder == null) return
         //空闲状态
         if (actionState != ItemTouchHelper.ACTION_STATE_IDLE) {
             ViewCompat.animate(viewHolder!!.itemView).setDuration(100).scaleX(1.2F).scaleY(1.2F)
                 .start()
         }
         super.onSelectedChanged(viewHolder, actionState)
 
     }
 
 
     /**
      * 滑动结束 清理选中得状态  恢复布局的样式
      */
     override fun clearView(recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder) {
         // 恢复显示
         // 这里不能用if判断，因为GridLayoutManager是LinearLayoutManager的子类，改用when，类型推导有区别
         ViewCompat.animate(viewHolder.itemView).setDuration(100).scaleX(1F).scaleY(1F).start()
         super.clearView(recyclerView, viewHolder)
     }
 
 
     /**
      * 是否支持长按拖拽，默认true
      * 因为我们外部监听了长安处理操作所以 这里需要禁用没改掉
      */
     override fun isLongPressDragEnabled(): Boolean {
         return false
     }
 
 
     /**
      * 是否支持侧滑默认true(线性布局的时候外部可以传进来)
      */
     override fun isItemViewSwipeEnabled(): Boolean {
         return isSwipe
     }
 }
 
 ```
 
 #### 3. ItemTouchHelper.Callback() 重要方法说明
 
 3.1 getMovementFlags():Int{}
  实现此方法获取移动的标志,方法内部需要调用makeMovementFlags(dragFlags, swipeFlags)方法设置拖拽和滑动的标志
  
  该标志定义每个状态下启用的移动方向
 
 ```
 
  // 这个方法用于让RecyclerView拦截向上滑动，向下滑动，想左滑动
     // makeMovementFlags(dragFlags, swipeFlags); dragFlags显示项目支持拖动的方向  swipe标记物品可以滑动的方向  0 表示禁止
     //  网格布局:上下左右
     //  线性布局:上下/左右
 
     override fun getMovementFlags(
         recyclerView: RecyclerView,
         viewHolder: RecyclerView.ViewHolder
     ): Int {
         //拖动标记 0 表示禁止
         var dragFlags = 0
         //侧滑标记
         var swipeFlags = 0
         var layoytManager = recyclerView.layoutManager
         if (layoytManager is GridLayoutManager) {
             //网格布局  默认禁止侧滑
             if (viewHolder.adapterPosition != mAdapter.limitPosition) {
                 dragFlags =ItemTouchHelper.LEFT or ItemTouchHelper.UP or ItemTouchHelper.RIGHT or ItemTouchHelper.DOWN
             }
             return makeMovementFlags(dragFlags, swipeFlags)
         } else if (layoytManager is LinearLayoutManager) {
 
             //处理 禁止滑动模块得逻辑  禁止移动得模块 禁止左右滑动
             if (viewHolder.adapterPosition != mAdapter.limitPosition) {
                 dragFlags = ItemTouchHelper.UP or ItemTouchHelper.DOWN
                 swipeFlags = ItemTouchHelper.START or ItemTouchHelper.END
             }
             return makeMovementFlags(dragFlags, swipeFlags)
         }
         //默认 禁止滑动  禁止拖拽
         return makeMovementFlags(dragFlags, swipeFlags)
 
     }
 
 ```
 
 3.2  makeMovementFlags(dragFlags, swipeFlags)
 
 创建移动标志的便捷方法,通过此方法创建移动的标志.dragflag显示项目可以拖动的方向,swipe标记物品可以滑动的方向。返回由给定拖动和滑动标志组成的整数。
 ```
 //源码
  /**
  
          * Convenience method to create movement flags.
          * <p>
          * For instance, if you want to let your items be drag & dropped vertically and swiped
          * left to be dismissed, you can call this method with:
          * <code>makeMovementFlags(UP | DOWN, LEFT);</code>
          *
          * @param dragFlags  The directions in which the item can be dragged.
          * @param swipeFlags The directions in which the item can be swiped.
          * @return Returns an integer composed of the given drag and swipe flags.
          */
         public static int makeMovementFlags(int dragFlags, int swipeFlags) {
             return makeFlag(ACTION_STATE_IDLE, swipeFlags | dragFlags)
                     | makeFlag(ACTION_STATE_SWIPE, swipeFlags)
                     | makeFlag(ACTION_STATE_DRAG, dragFlags);
         }
 
 ```
 
 3.3 override fun onMove( recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder,target: RecyclerView.ViewHolder): Boolean {}
 
 Item拖动的项从其旧位置移动到新位置时调用onMove()方法,在这里我们可以处理移动数据的刷新(重新排序数据)
 
 ```
     /**
      * 拖拽移动得回调
      */
     override fun onMove(
         recyclerView: RecyclerView,
         viewHolder: RecyclerView.ViewHolder,
         target: RecyclerView.ViewHolder
     ): Boolean {
         // 起始位置
         val fromPosition = viewHolder.adapterPosition
         // 结束位置
         val toPosition = target.adapterPosition
         // 固定位置  处理笃定位置不要移动
         if (fromPosition == mAdapter.limitPosition || toPosition == mAdapter.limitPosition) {
             return false
         }
         // 根据滑动方向 交换数据  for 循环不包含  toPosition  1 替代2     2替代3  3 替代4
         if (fromPosition < toPosition) {
             // 含头不含尾
             for (index in fromPosition until toPosition) {
                 //交换集合得位置
                 Collections.swap(mData, index, index + 1)
             }
         } else {
             // 含头不含尾
             for (index in fromPosition downTo toPosition + 1) {
                 Collections.swap(mData, index, index - 1)
             }
         }
         // 刷新布局
         mAdapter.notifyItemMoved(fromPosition, toPosition)
         return true
     }
     
 ```
 
 3.4 拖拽/滑动状态的处理
 
 ```
     /**
      * 选中会回调这里  处理选中布局的展示样式
      */
     override fun onSelectedChanged(viewHolder: RecyclerView.ViewHolder?, actionState: Int) {
 
         if (viewHolder == null) return
         //空闲状态
         if (actionState != ItemTouchHelper.ACTION_STATE_IDLE) {
             ViewCompat.animate(viewHolder!!.itemView).setDuration(100).scaleX(1.2F).scaleY(1.2F)
                 .start()
         }
         super.onSelectedChanged(viewHolder, actionState)
 
     }
 
 
     /**
      * 滑动/拖拽结束 清理选中得状态  恢复布局的样式
      */
     override fun clearView(recyclerView: RecyclerView, viewHolder: RecyclerView.ViewHolder) {
         // 恢复显示
         // 这里不能用if判断，因为GridLayoutManager是LinearLayoutManager的子类，改用when，类型推导有区别
         ViewCompat.animate(viewHolder.itemView).setDuration(100).scaleX(1F).scaleY(1F).start()
         super.clearView(recyclerView, viewHolder)
     }
 
 ```
 
 
 *注意:*
 
 actionState表示Item的状态
 
 - ItemTouchHelper.ACTION_STATE_IDLE 空闲状态。
 
 
 
 - ItemTouchHelper.ACTION_STATE_SWIPE 滑动状态。
 
 
 
 - ItemTouchHelper.ACTION_STATE_DRAG 拖拽状态。
 
 3.5 滑动/拖拽的开关
 
 ```
   /**
      * 是否支持长按拖拽，默认true
      * 因为我们外部监听了长安处理操作所以 这里需要禁用没改掉
      */
     override fun isLongPressDragEnabled(): Boolean {
         return false
     }
 
 
     /**
      * 是否支持侧滑默认true(线性布局的时候外部可以传进来)
      */
     override fun isItemViewSwipeEnabled(): Boolean {
         return isSwipe
     }
 
 ```
 
 ## 总结
 
 
 ItemTouchHelper是一个实用系统类，用于向RecyclerView添加滑动解除和拖放支持.通过这个类通过ItemTouchHelper.Callback() 回调帮助开发者实现了拖拽和滑动的效果
 
 写作不易欢迎点赞
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 
 


 








