<template>
<div class="tabbar-wrap">
    <!-- banner -->
    <c-top-back :mar='true'></c-top-back>
    <c-messages v-if="product_no" :status="2" msg="商品不存在或已删除">
        <div slot="btn">
          <c-button type="primary" text="返回首页" @click.native="btnClick"></c-button>
        </div>
    </c-messages>
    <div v-else>
        <div v-if="dataReady">
            <c-product-wrap type="normal">
              <c-product :title="goods.name" :des="logistics_fee1=='0.00'?'包邮':'运费：¥'+logistics_fee1" :price="goods.price" :pic="goods.pics"></c-product>
            </c-product-wrap>
            <c-cell-wrap>
                <c-cell :title="storeInfo.name" value="进入店铺" is-link @click.native="link_home">
                    <img class="wq-img" slot="icon" src='http://static.taggole.com/sithbrobot/poster/1489997146660.jpg'/>
                </c-cell>
            </c-cell-wrap>

            <c-title title="商品详情"></c-title>
            <div class="wrap-detail">
                <p>{{goods.product_detail}}</p>
                <p v-for='i in goods.product_desc'><img v-bind:src="i" alt="" style="width:100%; height:auto;"></p>
            </div>

            <c-sku v-model="sku.status" fillable :title="goods.name" :img="goods.pics" :price="goods.price" :text="sku.buttonTxt" :sku='skuList' @submit="submit"></c-sku>
            <c-button text="立即购买" size="block" @click.native="barClick" type="primary" style="position:fixed; bottom:0"></c-button>
        </div>
    </div>
        
</div>

</template>
<script>
    import utils from '../../libs/utils.js';
    import constants from '../../libs/constants.js';
    export default {
        data (){
            return {
                dataReady:false,
                goods:{},//商品信息
                skuList:[],//商品sku列表
                skuList1:[],
                storeInfo:{name:localStorage.getItem('store_name')},//店铺信息
                content:{},//商品详细信息
                goods_id:0,//商品id
                sku:{
                    status:false,
                    url:basepath + "/mall/store/shelf_product_sku_get",
                    buttonTxt:"下一步",
                },
                submitType:"buy",
                hasCart:true,//是否显示购物车
                limitNum:'',
                rule_flag:'',
                pageId:'',
                invoiceTypes:[],
                logistic_flg:1,
                product_no:false,
                logistics_fee1:0
            }
        },
        mounted: function () {
            this.$nextTick(function () {  
                this.$vux.alert.hide();
                this.init();
            })
        },
        methods:{
            submit(submitData){
                if(this.submitType == "buy"){
                    var buy_info_price = null;
                    $.each(this.skuList1,function(i,v){
                        if(v.id==submitData.sku.id){
                            buy_info_price = v.price;
                        }
                    })
                    
                    var buy_info = {
                        "products":[
                            {
                                "id":submitData.sku.id,
                                "name": this.goods.name,//商品名称
                                "pic": this.goods.pics,//商品图片
                                "properties": submitData.sku.name,//规格信息
                                "price": buy_info_price,//货架商品价格
                                "num": submitData.num,//购买商品的数量
                            }
                        ],
                        'logistics_fee':this.goods.logistics_fee,
                        'discount':0,
                        'logistic_flg':1
                    };
                    localStorage.setItem("buy_info",JSON.stringify(buy_info));
                    localStorage.removeItem('ORDERID');
                    utils.go({path:"/order/add"},this.$router);
                }else{
                    this.sku.status = false;
                    this.$vux.toast.show({
                        text:"操作成功",
                        type:"success"
                    });
                }


            },
            barClick(type){
                this.sku.status = true;
                this.submitType = "buy";
            },
            link_home(){
                utils.go({path:'/home/home',query:{id:this.$route.query.store_id}},this.$router,true);
            },
            btnClick(){
                utils.go({path:'/home/home',query:{id:this.$route.query.store_id}},this.$router,true);
            },
            init(){
                let that = this;
                utils.ajax({
                    url: "/user/product/get",
                    async:false,
                    data:{"id":that.$route.query.product_id},
                    success: function(data){
                        if(data.code=='SUCCESS'){
                            that.goods=data.result;
                            that.goods.logistics_fee=that.goods.logistics_fee;
                            that.logistics_fee1=utils.formatPrice(that.goods.logistics_fee);
                            that.goods.product_desc=JSON.parse(that.goods.product_desc);
                            utils.ajax({
                                url: "/user/product/sku/get",
                                async:false,
                                data:{"product_id":data.result.id},
                                success: function(res){
                                    if(res.code=='SUCCESS'){
                                        var max=res.result[0].price*1;
                                        var min=res.result[0].price*1;
                                        var len=res.result.length;

                                        for(var ii=1; ii<len; ii++){
                                          if(res.result[ii].price*1 > max){
                                            max = res.result[ii].price*1;
                                          }
                                        }

                                        for(var i=1; i<len; i++){
                                          if(res.result[i].price*1 < min){
                                            min = res.result[i].price*1;
                                          }
                                        }
                                        if(min!=max){
                                            that.goods.price='¥'+utils.formatPrice(min)+' ~ ¥'+utils.formatPrice(max);
                                        }else{
                                            that.goods.price='¥'+utils.formatPrice(min);
                                        }
                                        that.skuList1=res.result;
                                        $.each(res.result,function(i,v){
                                            that.skuList.push({
                                                id:v.id,
                                                name:v.sku_str,
                                                price:utils.formatPrice(v.price),
                                                stock:v.stock
                                            })
                                        })
                                    }else{
                                        that.$vux.alert.show(res.message);
                                    }
                                }
                                
                            });

                            that.dataReady = true;
                        }else if(data.code=='xvp_product1002'){
                            that.product_no=true;
                        }else if(data.code=='user_seller_error'){
                            that.$vux.alert.show({content:'访问超时',onHide :function(){
                              utils.go({path:'/home/home',query:{id:that.$route.query.store_id}},that.$router);
                            }});
                        }else{
                            that.$vux.alert.show(data.message);
                        }
                        // else if(data.code=='user_seller_error'){
                        //     utils.ajax({
                        //       url: basepath + "/user/user/getIsvInfo",
                        //       success: function(data) {
                        //         if(data.code=="SUCCESS") {  

                        //           $xvp.login({
                        //               app_key : data.result.appId,
                        //               isv_url: data.result.isvUrl,
                        //               success : function(xvp_uid){
                        //                 utils.ajax({
                        //                   url: basepath + "/user/user/login",
                        //                   data:{'xvp_uid':xvp_uid},
                        //                   success: function(res) {
                        //                     if(res.code=="SUCCESS") { 
                        //                       that.init();

                        //                     } else {
                        //                       that.$vux.alert.show(res.message);
                        //                     }
                        //                   },
                        //                 });
                        //               }
                        //           });

                        //           } else {
                        //             that.$vux.alert.show(data.message);
                        //           }
                        //         },
                        //       });
                        // }

                    } 
                });
                utils.MenuShare();
                // utils.ajax({
                //   url:basepath+ "/common/wxconfig/get",
                //   data:{'base_url':window.location.href.split('#')[0]},
                //   success: function(data) {
                //     if(data.code=='SUCCESS') {
                //       wx.config({
                //           debug: false, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
                //           appId: data.result.appId, // 必填，公众号的唯一标识
                //           timestamp: data.result.timestamp, // 必填，生成签名的时间戳
                //           nonceStr: data.result.nonceStr, // 必填，生成签名的随机串
                //           signature: data.result.signature,// 必填，签名，见附录1
                //           jsApiList: ['onMenuShareAppMessage','onMenuShareTimeline'] // 必填，需要使用的JS接口列表，所有JS接口列表见附录2
                //       });
                //       wx.ready(function(){
                //         wx.onMenuShareAppMessage({
                //             title: that.goods.title, // 分享标题
                //             desc: '【'+that.storeInfo.name+'】发现好商品，立即分享给你，进店有惊喜呦。', // 分享描述
                //             link: window.location.href, // 分享链接
                //             imgUrl: that.goods.pics // 分享图标
                //         });

                //         wx.onMenuShareTimeline({
                //             title: that.goods.title, // 分享标题
                //             link: window.location.href, // 分享链接
                //             imgUrl: that.goods.pics // 分享图标
                //         });
                //       })
                //     } else {
                //       that.$vux.alert.show(data.message);
                //     }
                //   },
                // });
            }
            
        },
        components:{
            "cProduct": require('../../components/x-product/x-product.vue'),
            "cProductWrap": require('../../components/x-product/x-product-wrap.vue'),
            "cTopBack":require('../../components/x-top-back/x-top-back.vue'),
            "cCellWrap":require('../../components/cell/cell-wrap.vue'),
            "cCell":require('../../components/cell/cell.vue'),
            "cButton":require('../../components/button/button.vue'),
            "cTitle":require('../../components/x-title/x-title.vue'),
            "cPayBar1":require('../../components/x-pay-bar/x-pay-bar1.vue'),
            "cPayBar1":require('../../components/x-pay-bar/x-pay-bar2.vue'),
            "cSku":require('../../components/x-sku/x-sku.vue'),
            "cMessages": require('../../components/x-messages/x-messages.vue'),
        }
    }
</script>
<style>
    .tabbar-wrap .wq-img{
        width:20px; margin-right:10px;
    }
    .tabbar-wrap .weui_cell_ft.with_arrow {
        margin-right:0 !important;
    }
    .tabbar-wrap .weui_cell_ft.with_arrow:after{
        right:0 !important;
    }
    .tabbar-wrap .pdt-tit{
      display: -webkit-box;
          overflow: hidden;
          text-overflow: ellipsis;
          -webkit-line-clamp: 2;
          -webkit-box-orient: vertical;
    }
    .tabbar-wrap .weui_cell_bd{
            overflow: hidden !important;
    }
    .tabbar-wrap .weui_cell_bd div{
        width: 55% !important;
            overflow: hidden !important;
            text-overflow: ellipsis !important;
            white-space: nowrap !important;
    }
</style>
