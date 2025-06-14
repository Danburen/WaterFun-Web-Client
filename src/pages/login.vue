<script setup lang="ts">
import axios from "axios";
import request from "~/utils/requests/axiosRequest";
import {ref, reactive, onMounted, onBeforeMount, nextTick} from "vue";
import {ElMessage, type FormInstance, type FormRules, type TabsPaneContext} from "element-plus";
import { deBounce , throttle} from "@/utils/TriggerControl.js"
import VerifyingCodeButton from "~/components/auth/VerifyingCodeButton.vue";
import AuthBox from "~/components/auth/authBox.vue";
import type { LoginRequest , LoginType } from '@/types/LoginRequest'
import { validateAuthname , validatePassword , validateVerifyCode} from "~/utils/Validator";
import type {ElInput} from "../../.nuxt/components";
import { Loading } from "@element-plus/icons-vue"
import {undefined} from "zod";
import {ErrorMessage} from "vee-validate";
import {useApiFetch} from "~/utils/requests/useFetchRequest";
import type {ErrorCode} from "~/types/ErrorCode";

type LoginTabType = 'password'|'fast-auth';

const i18n = useI18n();
const router = useRouter();

const passAuthForm = ref<FormInstance>();
const fastAuthForm = ref<FormInstance>();
const loginTab = ref<LoginTabType>('password');
const captchaLoading = ref(false);

const buttonLoad = ref(false)

const captchaImage = useState('captchaImage', () => '');


const passLoginForm = reactive({
  username:'',
  password:'',
  captcha:'',
  loginType: 'password' as LoginType,
})

const fastLoginForm = reactive({
  username:'',
  verifyCode:'',
  loginType: 'sms' as LoginType,
})

const passAuthRules = reactive<FormRules<typeof passLoginForm>>({
  username:[{validator: validateAuthname('password'),trigger: "blur" }],
  password:[{validator: validatePassword(false),trigger: "blur"}],
  captcha:[{validator: validateVerifyCode(false), trigger: "blur"}],
})

const fastAuthRules = reactive<FormRules<typeof fastLoginForm>>({
  username: [{validator: validateAuthname('sms'),trigger: "blur" }],
  verifyCode: [{validator: validateVerifyCode(false), trigger: "blur"}],
})

const resetForm = (passAuthForm: FormInstance | undefined,fastAuthForm: FormInstance | undefined) => {
  if(!passAuthForm) return;
  passAuthForm.resetFields();
  if(!fastAuthForm) return;
  fastAuthForm.resetFields();
}

const buildRequest= ():LoginRequest =>{
  if(loginTab.value === "password"){
    return passLoginForm as LoginRequest;
  }else{
    return {
      username: fastLoginForm.username,
      loginType: fastLoginForm.loginType,
      ...(fastLoginForm.loginType === 'sms'
              ? { smsCode: fastLoginForm.verifyCode }
              : { emailCode: fastLoginForm.verifyCode }
      ),
    }as LoginRequest;
  }
}

const submitForm = (form:FormInstance | undefined) => {
  if(!form) return;
  form.validate((valid)=>{
    buttonLoad.value = true;
    if(valid){
      request.post('auth/login',buildRequest()).then(res=>{
        console.log(res.data)
        ElMessage({
          message: i18n.t('message.success.login-success'),
          type: "success"
        })
        router.push('index');
      }).catch(err=>{
        console.log(err.data.message);
        ElMessage({
          message: getErrorMessage(err.data.code),
          type: "error"
        })
        switch (err.data.code) {
          case 40004:
          case 40006:
          case 40007: refreshCaptcha();break;
        }
      }).finally(()=>{
        buttonLoad.value = false;
      })
    }else{
      console.error('error occurred when submit form');
    }
  })
}

watch(()=>fastLoginForm.username,deBounce((value:string)=>{
  if(loginTab.value === 'fast-auth'){
      if(value.includes('@')){
        fastLoginForm.loginType = 'email';
        fastAuthRules.username = [{validator: validateAuthname('email'),trigger: "blur" }];
      }else{
        fastLoginForm.loginType = 'sms';
        fastAuthRules.username = [{validator: validateAuthname('sms'),trigger: "blur" }];
      }
    }
},300))

const refreshCaptcha = throttle(()=>{
  if(! captchaLoading.value){ 
    captchaLoading.value = true;
    request.get('/auth/captcha',{responseType: 'arraybuffer',meta:{ needsCSRF: false }}).then(res=>{
      const base64 = convertArrayBufferToBase64(res.data);
      captchaImage.value = `data:image/jpeg;base64,${base64}`;
    }).catch(err=>{
      ElMessage.error(i18n.t('message.error.api-error'));
      console.log(err);
    }).finally(()=>{
      captchaLoading.value = false;
    })
  }
},1000,()=>{
  ElMessage.error(i18n.t('message.throttled.click-too-fast'));
})

onBeforeMount(() => {
  refreshCaptcha();
});
</script>

<template>
    <auth-box>
      <el-tabs
          type="border-card"
          :stretch="true"
          v-model="loginTab"
          @tab-click="resetForm(passAuthForm,fastAuthForm)"
          class="login-tab"
      >
        <!--密码登录-->
        <el-tab-pane :label="$t('auth.tabs.password')" name="password">
          <el-form
              ref = 'passAuthForm'
              label-width="auto"
              :model="passLoginForm"
              :rules="passAuthRules"
              class="auth-form"
          >
            <el-form-item :label="$t('auth.username')" prop="username">
              <el-input
                  v-model="passLoginForm.username"
                  :placeholder="$t('auth.placeholder.username')"
                  class="login-input"
              ></el-input>
            </el-form-item>
            <el-form-item :label="$t('auth.password')"  prop="password">
              <el-input
                  v-model="passLoginForm.password"
                  type="password"
                  :placeholder="$t('auth.placeholder.password')"
                  class="login-input"
                  show-password
              ></el-input>
            </el-form-item>
            <!--验证码-->
            <el-form-item :label="$t('auth.verify-code')" prop="captcha">
              <div class="captcha-container">
                <el-input v-model="passLoginForm.captcha" :placeholder="$t('auth.placeholder.verify-code')" style="width: 60%" prop="verifyCode"></el-input>
                  <el-image v-loading="captchaLoading" v-if="captchaImage" :src="captchaImage" @click="()=>{ refreshCaptcha();}" class="captcha-image" alt="Captcha"/>
                  <el-skeleton v-else :rows="1" animated style="width: 120px; height: 20px" />
              </div>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" class="login-btn" @click="submitForm(passAuthForm)">{{ $t('auth.btn.login') }}</el-button>
              <div class="password-addition">
                <el-button size="small" link>{{ $t('auth.forget-password') }}</el-button>
                <el-button size="small" link class="to-register" @click.prevent="router.push('/register')" >{{ $t('auth.to-register') }}</el-button>
              </div>
            </el-form-item>
          </el-form>
        </el-tab-pane>
        <!--快捷登录-->
        <el-tab-pane :label="$t('auth.tabs.email-phone')" name="fast-auth">
          <el-form
              ref = 'fastAuthForm'
              label-width="auto"
              :model="fastLoginForm"
              :rules="fastAuthRules"
              class="auth-form"
          >
            <el-form-item :label="$t('auth.email') + '/' + $t('auth.phone')" prop="username">
              <el-input
                  ref='usernameInputRef'
                  v-model="fastLoginForm.username"
                  :placeholder="$t('auth.placeholder.email-phone')"
                  class="login-input"
              ></el-input>
            </el-form-item>
            <el-form-item :label="$t('auth.verify-code')" prop="verifyCode">
              <el-input
                  v-model="fastLoginForm.verifyCode"
                  :placeholder="$t('auth.placeholder.verify-code')"
                  class="login-input"
              >
                <template #append>
                  <VerifyingCodeButton></VerifyingCodeButton>
                </template>
              </el-input>
            </el-form-item>
            <el-form-item>
              <el-button
                  type="primary"
                  class="login-btn"
                  @click="submitForm(fastAuthForm)"
                  :loading = "buttonLoad"
              >{{ $t('auth.btn.login') }} / {{$t('auth.btn.register')}}</el-button>
            </el-form-item>
          </el-form>
        </el-tab-pane>
      </el-tabs>
    </auth-box>
</template>

<style scoped>

.auth-form{
  font-size: small;
  margin-top: 20px;
}

.password-addition{
  display: flex;
  align-items: center;
  width: 100%;
}

.password-addition .to-register{
  margin-left: auto;
}

.login-btn{
  width: 100%;
  margin: 5px auto;
}

.captcha-container {
  display: flex;
  align-items: center;
  width: 100%;
}
:deep(.el-tabs__content) {
  padding-bottom: 0;
}
</style>