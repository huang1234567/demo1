<!--
 * 创建人: 黄雨红
 * 创建时间: 2020-04-17
 * 描述: 登录页面
 -->
<template>
  <div class="loginPage">
    <div class="formBox">
      <van-form @submit="onSubmit">
        <van-field
          v-model="userInfo.loginName"
          name="账号"
          label="账号"
          placeholder="请输入账号"
          clearable
          @blur="inputBlur"
        />
        <van-field
          v-model="userInfo.password"
          type="password"
          name="密码"
          label="密码"
          placeholder="请输入密码"
          clearable
          @blur="inputBlur"
        />
        <van-field
          v-model="userInfo.rand"
          type="password"
          name="验证码"
          label="验证码"
          placeholder="验证码"
          clearable
          ref="vertifiCode"
          v-show="verifyCodeShow"
          @blur="inputBlur"
        />
        <div class="codeimg" id="v_codeimg" ref="codeimg" v-show="verifyCodeShow"></div>
        <div style="margin:25px 20px;">
          <van-button round block type="info" native-type="submit">登录</van-button>
        </div>
        <van-loading size="30px" v-show="loadingShow" class="textCenter">加载中...</van-loading>
      </van-form>
    </div>
    <van-overlay :show="loadingShow" />
  </div>
</template>

<script>
import Storage from "@/components/util/utilStorage.js";
import "@/plugin/gVerify.js";
import { Dialog } from "vant";
import router, { createRouter } from "@/router";
import { getToken, getCityCodeId, getPurviewCity ,getJsTicket } from "@/request/index.js";
export default {
  components: {},
  data() {
    return {
      userInfo: {
        loginName: "root",
        password: "123456",
        userId: "",
        rand: "",
        openId: ""
      },
      showKeyboard: true,
      verifyCode: "",
      verifyCodeShow: false,
      failLog: 0,
      loadingShow: false //加载提示是否显示
    };
  },
  created() {
    
  },
  mounted() {
    this.init();
    Storage.localSet("loginName", "");
    Storage.sessionRemove("mytoken"); //token移除
    this.verifyCode = new GVerify("v_codeimg");
  },
  methods: {
    init() {
      getJsTicket().then(res=>{
        console.log('22222222',res)
      })
    },
    onSubmit(values) {
      if (
        this.userInfo.loginName.trim() == "" ||
        this.userInfo.password.trim() == ""
      ) {
        Dialog.alert({
          message: "请输入账号或密码！"
        });
        return false;
      }
      if (
        this.verifyCodeShow &&
        !this.verifyCode.validate(this.userInfo.rand)
      ) {
        Dialog.alert({
          message: "请正确输入验证码！"
        });
        return false;
      }
      let userParams = {
        userName: this.userInfo.loginName,
        password: this.userInfo.password,
        type: "2"
      };
      this.loadingShow = true;
      getToken(userParams)
        .then(res => {
          if (res.code == "2000") {
            let userData = res.data;
            this.userInfo.userId = userData.userId;
            Storage.localSet("loginName", this.userInfo.loginName);
            Storage.localSet("userId", this.userInfo.userId);
            this.$store.commit("setUserInfo", this.userInfo.loginName);
            Storage.sessionSet("mytoken", res.data.token);
            if (res.data.roleData) {
              Storage.localSet("roleList", res.data.roleData);
              this.$store.commit("setRoleData", res.data.roleData); //路由权限数据
              let rolenew = this.$store.getters.getRoleData;
              rolenew.push({
                path: "*",
                component: () => import("@/components/login")
              });
              console.log("路由数据", rolenew);
              router.matcher = createRouter().matcher;
              router.addRoutes(rolenew); // routes 为登录后后台接口返回的动态路由
            }
            // 公司主体接口改为组织权限接口-2020/7/7 hyh注释
            // getCityCodeId()
            //   .then(res => {
            //     Storage.localSet("CityCodeLists", res.data);
            //     this.$router.push({ path: "/operationHome" });
            //   })
            //   .catch(function(err) {
            //     console.log(err);
            //   });
            getPurviewCity({ userId: this.userInfo.userId })
              .then(res => {
                if (res.code == "2000") {
                  // console.log(res)
                  //1、维度城市-先取amout=='1'，根据城市去重，原来城市数据：{aliasName: "集团",orgId: "X01",orgName: "新力地产",parentId: "X02"}
                  // 2、项目查询页面-根据勾选的城市项目，才能显示对应的城市项目
                  let purviewData = res.data;
                  let cityData = [];//维度城市数据
                  let itemData = [];//单项目城市数据
                  purviewData.map(item => {
                  let cityObj = {};//维度城市数据
                  let itemObj = {};//单项目城市数据
                  //单项目城市显示的数据
                   if(item.amount == "1"){
                       itemObj = item;
                      itemObj["aliasName"] = item.parentname.split("城市公司")[0];
                      itemData.push(itemObj);
                    }
                  // 维度城市显示的数据
                    if ((item.objectname=="所有项目"&&item.amount == "1")||item.parentname=="新力地产"&&item.amount == "1") {
                      cityObj = item;
                      cityObj["aliasName"] = item.parentname.split("城市公司")[0];
                      cityData.push(cityObj);
                    }
                  });
                  let mapObj = new Map();//维度城市
                  let setCityList = cityData.filter(
                    a =>
                      !mapObj.has(a.parentname) && mapObj.set(a.parentname, 1)
                  );
                  let mapItemObj=new Map();//单项目城市
                   let setItemList = itemData.filter(
                    a =>
                      !mapItemObj.has(a.parentname) && mapItemObj.set(a.parentname, 1)
                  );
                  // cityData.push({aliasName: "集团",orgId: "X01",orgName: "新力地产",parentId: "X02"})
                  Storage.localSet("CityCodeLists", setCityList);
                  Storage.localSet("itemCodeLists", setItemList);
                  this.$router.push({ path: "/home" });
                  console.log("维度城市数据", setCityList,"单项目城市数据",setItemList);
                }
              })
              .catch(function(err) {
                console.log(err);
              });
            // this.$router.push({ path: "/operationHome" });
          } else {
            Dialog.alert({
              message: res.message
            });
            let countNums = this.failLog;
            this.failLog = 1 + (countNums - 0);
            if (this.failLog >= 3) {
              //显示验证码
              this.verifyCode.refresh();
              // this.verifyCodeShow=true; 验证码暂时不放开
            }
          }
          this.loadingShow = false; //关闭loading
        })
        .catch(error => {
          this.loadingShow = false; //关闭loading
          let countNums = this.failLog;
          this.failLog = 1 + (countNums - 0);
          if (this.failLog >= 3) {
            this.verifyCode.refresh();
            // this.verifyCodeShow=true; 验证码暂时不放开
          }
        });
      //   console.log("submit", values);
    },
    inputBlur(event) {
      //   console.log(event);
      //   if (event.target.value.trim() == "") {
      //       Dialog.alert({
      //         message: event.target.placeholder,
      //         }).then(() => {
      //         return;
      //         });
      //   }
    }
  },
  destroyed() {}
};
</script>

<style lang="scss" scoped>
.loginPage {
  padding: 10px;
  height: 100%;
  // background-color: #fff;

  .textCenter {
    text-align: center;
  }
  .formBox {
    margin-top: 30px;
    // border: 1px solid #f2f2f2;
    // background-color: #fff;
    height: 50%;
    padding: 10px;
    border-radius: 20px;
    .codeimg {
      float: right;
      margin-top: -38px;
      display: block;
      width: 100px;
      height: 34px;
      top: 180px;
      // display: none;
      position: absolute;
      right: 6%;
    }
  }
}
</style>
