<template>
  <div class="login-container">
    <logo />
    <el-form
      class="login-form"
      label-width="90px"
      :model="form"
      :rules="rules"
      ref="loginForm"
    >
      <el-form-item prop="email" label="邮箱">
        <el-input v-model="form.email" placeholder="请输入邮箱"></el-input>
      </el-form-item>
      <el-form-item prop="captcha" label="验证码" class="captcha-container">
        <div class="captcha">
          <img :src="code.captcha" alt="" @click="resetCaptcha" />
        </div>
        <el-input v-model="form.captcha" placeholder="请输入验证码"></el-input>
      </el-form-item>

      <el-form-item
        prop="emailcode"
        label="邮箱验证码"
        class="captcha-container"
      >
        <div class="captcha">
          <el-button
            @click="sendEmailCode"
            :disabled="send.timer > 0"
            type="primary"
            >{{ sendText }}</el-button
          >
        </div>
        <el-input
          v-model="form.emailcode"
          placeholder="请输入邮箱验证码"
        ></el-input>
      </el-form-item>

      <el-form-item prop="passwd" label="密码">
        <el-input
          v-model="form.passwd"
          placeholder="请输入密码"
          type="password"
        ></el-input>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" @click.native.prevent="handleLogin"
          >登录</el-button
        >
      </el-form-item>
    </el-form>
  </div>
</template>
<script>
import md5 from "md5";
import Logo from "~/components/Logo.vue";
export default {
  components: {
    Logo
  },
  layout: "login",
  methods: {
    resetCaptcha() {
      this.code.captcha = "/api/captcha?_t=" + new Date().getTime();
    },

    async sendEmailCode() {
      await this.$http.get("/sendcode?email=" + this.form.email);

      this.send.timer = 10;
      this.timer = setInterval(() => {
        this.send.timer -= 1;

        if (this.send.timer === 0) {
          clearInterval(this.timer);
        }
      }, 1000);
    },
    handleLogin() {
      this.$refs.loginForm.validate(async valid => {
        if (valid) {
          let obj = {
            email: this.form.email,
            passwd: md5(this.form.passwd),
            captcha: this.form.captcha,
            emailcode: this.form.emailcode
          };
          let ret = await this.$http.post("/user/login", obj);
          console.log(ret);

          if (ret.code == 0) {
            this.$message.success("登录成功");

            localStorage.setItem("token", ret.data.token);

            setTimeout(() => {
              this.$router.push("/");
            }, 500);
          } else {
            this.$message.error(ret.message);
          }
        } else {
          console.log("校验失败");
        }
      });
    }
  },
  computed: {
    sendText() {
      if (this.send.timer <= 0) {
        return "发送";
      }

      return `${this.send.timer}s后发送`;
    }
  },
  data() {
    return {
      send: { timer: 0 },
      form: {
        email: "389538496@qq.com",
        captcha: "xyzd",
        passwd: "123456",
        emailcode: ""
      },
      rules: {
        email: [
          {
            required: true,
            message: "请输入邮箱"
          },
          {
            type: "email",
            message: "请输入正确的邮箱格式"
          }
        ],
        captcha: [
          {
            required: true,
            message: "请输入验证码"
          }
        ],
        emailcode: [
          {
            required: true,
            message: "请输入邮箱验证码"
          }
        ],
        passwd: [
          {
            required: true,
            pattern: /^[\w-]{6,12}$/g,
            message: "请输入6~12位密码"
          }
        ]
      },
      code: {
        captcha: "/api/captcha"
      }
    };
  }
};
</script>
<style></style>
