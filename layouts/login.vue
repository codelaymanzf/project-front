<template>
  <div>
    <nuxt />
  </div>
</template>
<script>
export default {};
</script>
<style>
.login-form {
  max-width: 800px;
  margin: 30px auto;
}
.login-container {
  width: 100%;
  min-height: 100%;
  text-align: center;
}

.captcha-container {
  width: 340px;
  position: relative;
}

.captcha-container .el-button {
  width: 100px;
}
.captcha-container .captcha {
  position: absolute;
  right: -110px;
}
.captcha-container .captcha img {
  display: block;
  width: 100px;
  height: 40px;
  cursor: pointer;
}
</style>
