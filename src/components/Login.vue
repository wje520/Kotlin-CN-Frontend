<template>
  <div class="dialog" v-if="showMode !==0">
    <div class="bg" v-on:click="hide"></div>
    <div class="cont">
      <section class="logo">
        <span><b>Kotlin</b> CHINA</span>
      </section>
      <div class="login" v-if="showMode === 1 || showMode === 3">
        <p v-if="showMode === 3">绑定GitHub账号到一个已创建的账号</p>
        <input v-model="nameInput" type="text" name="user" placeholder="用户名或邮箱"/>
        <span v-if="error && error.key == 'login-name'" class="error">{{ error.value }}</span>
        <input v-model="passwordInput" type="password" name="password" placeholder="用户密码"/>
        <span v-if="error && error.key == 'login-password'" class="error">{{ error.value }}</span>
        <button v-on:click="login" class="big-btn" v-if="showMode === 1">登录</button>
        <button v-on:click="login" class="big-btn" v-if="showMode === 3">确认基本信息</button>
        <div class="small-btn">
          <button v-on:click="switchMode" v-if="showMode === 1">还没有账号</button>
          <button v-on:click="switchMode" v-if="showMode === 3">或者创建全新的账号</button>
        </div>
      </div>
      <div class="register" v-if="showMode === 2 || showMode === 4">
        <p v-if="showMode === 4">首次使用GitHub账号登录，需要完善以下基本信息</p>
        <input v-model="emailInput" type="text" name="user" placeholder="邮箱"/>
        <span v-if="error && error.key == 'register-email'" class="error">{{ error.value }}</span>
        <input v-model="nameInput" type="text" name="user" placeholder="用户名"/>
        <span v-if="error && error.key == 'register-name'" class="error">{{ error.value }}</span>
        <input v-model="passwordInput" type="password" name="password" placeholder="密码"/>
        <span v-if="error && error.key == 'register-password'" class="error">{{ error.value }}</span>
        <input v-model="passwordRepeatInput" v-if="passwordInput.length >=8"
               type="password" name="password" placeholder="再次输入密码"/>
        <span v-if="error && error.key == 'register-password-repeat'" class="error">{{ error.value }}</span>
        <button v-on:click="register" class="big-btn" v-if="showMode === 2">注册并登录</button>
        <button v-on:click="register" class="big-btn" v-if="showMode === 4">创建并绑定</button>
        <div class="small-btn">
          <button v-on:click="switchMode" v-if="showMode === 2">已有账号，去登陆</button>
          <button v-on:click="switchMode" v-if="showMode === 4">绑定一个已有账号</button>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
  import Config from "../assets/js/Config.js";
  import Event from "../assets/js/Event.js";
  import Net from "../assets/js/Net.js";
  import LoginMgr from '../assets/js/LoginMgr';
  import Utils from '../assets/js/Util';
  import Cookie from 'js-cookie';

  export default {
    data () {
      return {
        showMode: 0,//0-none, 1-login, 2-register, 3-GitHub-login, 4-GitHub-register
        emailInput: '',
        nameInput: '',
        passwordInput: '',
        passwordRepeatInput: '',
        loginAlready: false,
        error: false
      }
    },
    created() {
      Event.on('request_login', (loginAlready) => {
        const githubToken = Cookie.get('X-App-Github');
        this.loginAlready = loginAlready;
        this.showMode = githubToken ? 3 : 1;
      });
      Event.on('request_register', (loginAlready) => {
        const githubToken = Cookie.get('X-App-Github');
        this.loginAlready = loginAlready;
        this.showMode = githubToken ? 4 : 2;
      });
    },
    watch: {
      showMode() {
        this.notifyError()
      },
      emailInput() {
        this.notifyError()
      },
      nameInput() {
        this.notifyError()
      },
      passwordInput() {
        this.notifyError()
      },
      passwordRepeatInput() {
        this.notifyError()
      }
    },
    methods: {
      switchMode(){
        let githubToken = Cookie.get('X-App-Github');
        if (!githubToken) {
          this.showMode = this.showMode === 1 ? 2 : 1
        } else {
          this.showMode = this.showMode === 3 ? 4 : 3
        }
      },
      forget(){

      },
      notifyError() {
        if (this.showMode === 1 || this.showMode === 3) {
          if (this.nameInput.trim().length < 2) {
            this.error = {key: 'login-name', value: '请输入正确的用户名'}
          } else if (this.passwordInput.length < 8) {
            this.error = {key: 'login-password', value: '请输入正确的密码'}
          } else {
            this.error = false
          }
        } else if (this.showMode === 2 || this.showMode === 4) {
          if (!Utils.isValidEmail(this.emailInput)) {
            this.error = {key: 'register-email', value: '请输入正确的邮箱'}
          } else if (this.nameInput.trim().length < 2) {
            this.error = {key: 'register-name', value: '用户名过短'}
          } else if (this.passwordInput.length < 8) {
            this.error = {key: 'register-password', value: '密码需要8位或以上，仅限大小写与数字'}
          } else if (this.passwordRepeatInput !== this.passwordInput) {
            this.error = {key: 'register-password-repeat', value: '两次输入的密码不一致'}
          } else {
            this.error = false
          }
        }
      },
      login() {
        if (this.error) return;
        const githubToken = Cookie.get('X-App-Github');
        const githubUserInfo = Cookie.get('X-App-Github-User');
        Net.post({
          url: Config.URL.account.login,
          condition: githubToken ? {
            'login_name': this.nameInput.trim(),
            'password': this.passwordInput,
            "github_token": githubToken,
            'logo': JSON.parse(githubUserInfo).github_avatar
          } : {
            'login_name': this.nameInput.trim(),
            'password': this.passwordInput,
          }
        }, (resp) => {
          LoginMgr.login(resp);
          this.hide()
        })
      },
      register() {
        if (this.error) return;
        const githubToken = Cookie.get('X-App-Github');
        const githubUserInfo = Cookie.get('X-App-Github-User');
        let name = this.nameInput;
        let email = this.emailInput;
        Net.post({
          url: Config.URL.account.register,
          condition: githubToken ? {
            "username": this.nameInput,
            "password": this.passwordInput,
            "email": this.emailInput,
            "github_token": githubToken,
            'logo': JSON.parse(githubUserInfo).github_avatar
          } : {
            "username": this.nameInput,
            "password": this.passwordInput,
            "email": this.emailInput,
          }
        }, (resp) => {
          console.log(resp.logo);
          LoginMgr.login({
            uid: resp.uid,
            token: resp.token,
            username: name,
            email: email,
            role: 0,
            logo: resp.logo
          });
          this.hide()
        })
      },
      hide() {
        Cookie.remove('X-App-Github');
        Cookie.remove('X-App-Github-User');
        if (this.showMode === 3 || this.showMode === 4) {
          setTimeout(() => window.location.href = '/', 300)
        }
        this.showMode = 0;
        const info = LoginMgr.info();
        if (this.loginAlready && info.isLogin) {
          this.loginAlready(info);
        }
        this.loginAlready = undefined;
        this.nameInput = '';
        this.emailInput = '';
        this.passwordInput = '';
        this.passwordRepeatInput = '';
        this.error = false;
      }
    }
  }
</script>

<style scoped>
  .dialog .cont {
    margin-left: calc(50% - 253px);
  }
</style>

<style scoped lang="less">
  .dialog {
    position: fixed;
    z-index: 3;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;
    .bg {
      position: absolute;
      background: #000000;
      filter: alpha(opacity=50);
      -moz-opacity: 0.5;
      opacity: 0.5;
      width: 100%;
      height: 100%;
    }
    .cont {
      position: absolute;
      width: 506px;
      margin-top: 5%;
      padding: 40px;
      box-sizing: border-box;
      background: white;
      border: 1px #f1f1f1 solid;
      box-shadow: 0 0 3px #2572e5;

      .logo {
        color: #6b6b6b;
        font-weight: bolder;
        font-size: 35px;
        text-align: center;
        b {
          color: #2b75e1;
        }
      }

      input {
        display: block;
        box-sizing: border-box;
        margin: 20px auto 10px auto;
        border: 1px #ddd solid;
        border-radius: 3px;
        outline: none;
        width: 100%;
        height: 60px;
        font-size: 20px;
        padding: 0 10px;
      }
      input:hover {
        -webkit-box-shadow: 0 0 3px #2e8ded;
      }
      input::-webkit-input-placeholder {
        color: #999;
      }
      input::-moz-placeholder {
        color: #999;
      }
      input:-webkit-autofill {
        -webkit-box-shadow: 0 0 0 1000px #fff inset;
      }
      .error {
        display: block;
        font-size: 14px;
        color: #fb6666;
        width: 100%;
        height: 20px;
      }
      .big-btn {
        margin: 30px auto 0 auto;
        display: block;
        width: 100%;
        height: 60px;
        background-color: #2572e5;
        color: white;
        font-size: 24px;
      }
      .big-btn:hover {
        background-color: #4599f7;
      }
      .big-btn:active {
        background-color: #1c4ecf;
      }
      .small-btn {
        margin: 4px auto 10px auto;
        width: 100%;
        button {
          color: #3989f8;
        }
        button:nth-child(2) {
          float: right;
        }
      }
    }

  }
</style>
