<template>
	<view class="videoContainer">
	  <!-- 头部区域 -->
	  <view class="header">
	    <image src="/static/images/video/video.jpg"></image>
	    <view class="search" bindtap="toSearch">
	      搜索音乐
	    </view>
	    <image src="/static/images/logo.png"></image>
	  </view>
	
	  <!-- 导航区域 -->
	  <scroll-view
	      scroll-x
	      class="navScroll"
	      enable-flex
	      :scroll-into-view="'scroll' + navId"
	      scroll-with-animation
	  >
	    <view :id="'scroll' + item.id" class="navItem" v-for="item in videoGroupList" :key="item.id">
	      <view class="navContent" :class="navId === item.id?'active': ''" @click="changeNav" :id="item.id" :data-id="item.id">
	        {{item.name}}
	      </view>
	    </view>
	  </scroll-view>
	
	  <!-- 视频列表区域 -->
	  <scroll-view
	      scroll-y
	      class="videoScroll"
	      refresher-enabled
	      @refresherrefresh="handleRefresher"
	      :refresher-triggered="isTriggered"
	      @scrolltolower="handleToLower"
	  >
	    <view class="videoItem" v-for="item in videoList" :key="item.id">
	      <video
	          :src="item.data.urlInfo.url"
	          @play="handlePlay"
	          :id="item.data.vid"
	          :poster="item.data.coverUrl"
	          class="common"
	          object-fit="cover"
	          v-if='videoId === item.data.vid'
	          @timeupdate="handleTimeUpdate"
	          @ended="handleEnded"
	      ></video>
	
	      <!-- 性能优化：使用image图片代替video标签 -->
	      <image v-else @click="handlePlay" :id="item.data.vid" class="common" :src="item.data.coverUrl"></image>
	
	
	      <view class="content">{{item.data.title}}</view>
	
	      <view class="footer">
	        <image class="avatar" :src="item.data.creator.avatarUrl"></image>
	        <text class="nickName">{{item.data.creator.nickname}}</text>
	        <view class="comments_praised">
	          <text class="item">
	            <text class="iconfont icon-buoumaotubiao15"></text>
	            <text class="count">{{item.data.praisedCount}}</text>
	          </text>
	          <text class="item">
	            <text class="iconfont icon-pinglun1"></text>
	            <text class="count">{{item.data.commentCount}}</text>
	          </text>
	          <button open-type="share" class="item btn">
	            <text class="iconfont icon-gengduo"></text>
	          </button>
	        </view>
	      </view>
	    </view>
	  </scroll-view>
	</view>

</template>

<script>
	import request from '../../utils/request'
	export default{
		data(){
			return{
				videoGroupList: [], // 导航标签数据
				    navId: '', // 导航的标识
				    videoList: [], // 视频列表数据
				    videoId: '', // 视频id标识
				    videoUpdateTime: [], // 记录video播放的时长
				    isTriggered: false, // 标识下拉刷新是否被触发
				    offset: 0,
				    index: 0,
			}
		},
		onLoad() {
			// 获取导航数据
			    this.getVideoGroupListData();
		},
		methods:{
			// 获取导航数据
			  async getVideoGroupListData(){
			    let videoGroupListData = await request('/video/group/list');
			    
			      this.videoGroupList= videoGroupListData.data.slice(0, 14),
			      this.navId= videoGroupListData.data[0].id
			  
			    // 获取视频列表数据
			    this.getVideoList(this.navId);
			  },
			  // 获取视频列表数据
			  async getVideoList(navId){
			    if(!navId){ // 判断navId为空串的情况
			      return;
			    }
			    let videoListData = await request('/video/group', {id: navId});
			    // 关闭消息提示框
			    uni.hideLoading();
			    
			    
			    let videoList = videoListData.datas.map(item => {
			      item.id = this.index++;
			      return item;
			    })
			
			    for (let i = 0; i < videoList.length; i++) {
			      let url = await request('/video/url', {id: videoList[i].data.vid});
			      videoList[i].data.urlInfo = url.urls[0]
			    }
			
			    
			   this.videoList = videoList,
			   this.isTriggered= false // 关闭下拉刷新
			  },
			
			  // 点击切换导航的回调
			  changeNav(event){
			    this.offset= 0
			    let navId = event.currentTarget.id; // 通过id向event传参的时候如果传的是number会自动转换成string
			    // let navId = event.currentTarget.dataset.id;
			    
			      this.navId= navId>>>0,
			      this.videoList = []
			    // 显示正在加载
			    uni.showLoading({
			      title: '正在加载'
			    })
			    // 动态获取当前导航对应的视频数据
			    this.getVideoList(this.navId);
			  },
			  
			  // 点击播放/继续播放的回调
			  handlePlay(event){
			    /*
			      问题： 多个视频同时播放的问题
			    * 需求：
			    *   1. 在点击播放的事件中需要找到上一个播放的视频
			    *   2. 在播放新的视频之前关闭上一个正在播放的视频
			    * 关键：
			    *   1. 如何找到上一个视频的实例对象
			    *   2. 如何确认点击播放的视频和正在播放的视频不是同一个视频
			    * 单例模式：
			    *   1. 需要创建多个对象的场景下，通过一个变量接收，始终保持只有一个对象，
			    *   2. 节省内存空间
			    * */
			    
			    let vid = event.currentTarget.id;
			    // 关闭上一个播放的视频
			    // this.vid !== vid && this.videoContext && this.videoContext.stop();
			    // if(this.vid !== vid){
			    //   if(this.videoContext){
			    //     this.videoContext.stop()
			    //   }
			    // }
			    // this.vid = vid;
			    // 更新data中videoId的状态数据
			      this.videoId= vid
			    // 创建控制video标签的实例对象
			    this.videoContext = uni.createVideoContext(vid);
			    // 判断当前的视频之前是否播放过，是否有播放记录, 如果有，跳转至指定的播放位置
			    let videoItem = this.videoUpdateTime.find(item => item.vid === vid);
			    if(videoItem){
			      this.videoContext.seek(videoItem.currentTime);
			    }
			    this.videoContext.play();
			    // this.videoContext.stop();
			  },
			  
			  // 监听视频播放进度的回调
			  handleTimeUpdate(event){
			    let videoTimeObj = {vid: event.currentTarget.id, currentTime: event.detail.currentTime};
			    /*
			    * 思路： 判断记录播放时长的videoUpdateTime数组中是否有当前视频的播放记录
			    *   1. 如果有，在原有的播放记录中修改播放时间为当前的播放时间
			    *   2. 如果没有，需要在数组中添加当前视频的播放对象
			    *
			    * */
			    let videoItem = this.videoUpdateTime.find(item => item.vid === videoTimeObj.vid);
			    if(videoItem){ // 之前有
			      videoItem.currentTime = event.detail.currentTime;
			    }else { // 之前没有
			      this.videoUpdateTime.push(videoTimeObj);
			    }
			  },
			  
			  // 视频播放结束调用的回调
			  handleEnded(event){
			    // 移除记录播放时长数组中当前视频的对象
			    this.videoUpdateTime.splice(this.videoUpdateTime.findIndex(item => item.vid === event.currentTarget.id), 1);
			  },
			  
			  // 自定义下拉刷新的回调： scroll-view
			  handleRefresher(){
			    console.log('scroll-view 下拉刷新');
			    // 再次发请求，获取最新的视频列表数据
			    this.getVideoList(this.navId);
			  },
			  
			  // 自定义上拉触底的回调 scroll-view
			  async handleToLower(){
			    console.log('scroll-view 上拉触底');
			    // 数据分页： 1. 后端分页， 2. 前端分页
			    console.log('发送请求 || 在前端截取最新的数据 追加到视频列表的后方');
			
			    let videoListData = await request('/video/group', {id: this.navId, offset: this.offset++});
			    console.log(videoListData)
			
			    let videoLists = this.videoList;
			
			    let videoList = videoListData.datas.map(item => {
			      item.id = this.index++;
			      return item;
			    })
			
			    for (let i = 0; i < videoList.length; i++) {
			      let url = await request('/video/url', {id: videoList[i].data.vid});
			      videoList[i].data.urlInfo = url.urls[0]
			    }
			
			    // 将视频最新的数据更新原有视频列表数据中
			    videoLists.push(...videoList);
			    this.videoList= videoLists
			  },
			  // 跳转至搜索界面
			  toSearch(){
			    wx.navigateTo({
			      url: '/pages/search/search'
			    })
			  }
		}
		
	}
</script>

<style>
.videoContainer .header {
  display: flex;
  padding: 10rpx;
}
.videoContainer .header image{
  width: 60rpx;
  height: 60rpx;
}

.videoContainer .header .search {
  border: 1rpx solid #eee;
  /*flex-grow: 可拉伸 flex-shrink： 可压缩 flex-basis: 当前元素的宽度*/
  /*flex默认值: flex-grow: 0, flex-shrink: 1, flex-basis: auto*/
  /*flex:1  flex-grow: 1, flex-shrink: 1, flex-basis: 0%*/
  /*flex:auto  flex-grow: 1, flex-shrink: 1, flex-basis: auto*/
  /*flex: 1会导致父元素宽度自动为100%*/
  flex: 1;
  margin: 0 20rpx;
  font-size: 26rpx;
  text-align: center;
  line-height: 60rpx;
  color: #d43c33;
}



/* 导航区域 */
.navScroll {
  display: flex;
  white-space: nowrap;
  height: 60rpx;
}


.navScroll .navItem {
	 display: inline-block;
  padding: 0 30rpx;
  font-size: 28rpx;
  height: 60rpx;
  line-height: 60rpx;
}
.navScroll .navContent {
  height: 60rpx;
  box-sizing: border-box;
}


.navItem .active {
  border-bottom: 1rpx solid #d43c33;
}


/* 视频列表 */
.videoScroll {
  margin-top: 10rpx;
  /* calc: 可以动态计算css的宽高， 运算符左右两侧必须加空格，否则计算会失效 */
  /* 视口单位： vh vw  1vh = 1%的视口高度  1vw = 1%的视口宽度*/
  height: calc(100vh - 152rpx);
  /*height: calc(100vh - 100rpx); 用来测试页面上拉触底*/
}
.videoItem {
  padding: 0 3%;
}
/*.videoItem video {*/
  /*width: 100%;*/
  /*height: 360rpx;*/
  /*border-radius: 10rpx;*/
/*}*/

.videoItem .common {
  width: 100%;
  height: 360rpx;
  border-radius: 10rpx;
}




.videoItem .content {
  font-size: 26rpx;
  height:80rpx;
  line-height: 80rpx;
  max-width: 500rpx;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

/* footer */
.footer {
  border-top: 1rpx solid #eee;
  padding: 20rpx 0;
}
.footer .avatar {
  width: 60rpx;
  height: 60rpx;
  border-radius: 50%;
  vertical-align: middle;
}

.footer  .nickName {
  font-size: 26rpx;
  vertical-align: middle;
  margin-left: 20rpx;
}

.footer .comments_praised {
  float: right;
}

.comments_praised .btn {
  display: inline;
  padding: 0;
  background-color: transparent;
  border-color: transparent;
}

.comments_praised .btn:after {
  border: none;
}

.comments_praised .item {
  margin-left: 50rpx;
  position: relative;
}

.comments_praised .item .count {
  position: absolute;
  top: -20rpx;
  font-size: 20rpx;
}








</style>
