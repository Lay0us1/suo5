<template>
  <div class="container">
    <n-card size="small">
      <n-form
          label-width="70"
          label-placement="left"
          size="medium"
          style="margin-bottom: -12px; margin-top:12px;"
      >
        <n-form-item label="目标" required>
          <n-input v-model:value="formValue.method" style="width: 85px; margin-right: 10px"/>
          <n-input v-model:value="formValue.target" placeholder="https://example.com/1.jsp"/>
          <n-button style="margin-left: 18px" :type="btnType" @click="runAction" :loading="runLoading">{{ btnName }}
          </n-button>
        </n-form-item>

        <n-form-item label="Socks5" style="margin-bottom: -24px" required>
          <n-space vertical class="full-width" :size="0">
            <n-input v-model:value="formValue.listen" placeholder="监听地址"/>
            <n-form inline>
              <n-form-item>
                <n-checkbox style="width: 80px"
                            @update:checked="authChange"
                            size="large">认证
                </n-checkbox>
              </n-form-item>
              <n-form-item>
                <n-input v-model:value="formValue.username"
                         :disabled="formValue.no_auth"
                         style="width: 210px"
                         placeholder="用户名"/>
              </n-form-item>
              <n-form-item>
                <n-input v-model:value="formValue.password"
                         :disabled="formValue.no_auth"
                         style="width: 210px"
                         placeholder="密码"/>
              </n-form-item>
            </n-form>
          </n-space>
        </n-form-item>

        <n-form-item label="模式选择">
          <n-space justify="space-between" id="mode">
            <n-radio-group v-model:value="formValue.mode">
              <n-radio value="auto">自动</n-radio>
              <n-radio value="full">全双工</n-radio>
              <n-radio value="half">半双工</n-radio>
            </n-radio-group>
            <n-button secondary style="float: right" strong @click="showAdvanced = true">高级配置</n-button>
          </n-space>
        </n-form-item>
      </n-form>
    </n-card>

    <n-alert :type="alertType" :bordered="false" style="margin-top: 24px">
      {{ alertContent }}
    </n-alert>

    <n-card title="运行日志" size="small" style="margin-top: 24px" embedded>
      <template #header-extra>
        <n-button text type="primary" @click="clearLog">
          <n-icon size="20">
            <svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" viewBox="0 0 32 32">
              <path d="M12 12h2v12h-2z" fill="currentColor"></path>
              <path d="M18 12h2v12h-2z" fill="currentColor"></path>
              <path d="M4 6v2h2v20a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V8h2V6zm4 22V8h16v20z" fill="currentColor"></path>
              <path d="M12 2h8v2h-8z" fill="currentColor"></path>
            </svg>
          </n-icon>
        </n-button>
      </template>
      <n-log :log="log" ref="logInst" :font-size="12" language="accesslog" :rows="18"/>
    </n-card>
    <n-space justify="space-between" style="margin-top:20px">
      <span>连接数: {{ status.connection_count }}</span>
      <span>CPU: {{ status.cpu_percent }}</span>
      <span>内存: {{ status.memory_usage }}</span>
      <span>版本: 0.5.0</span>
    </n-space>

    <div class="footer">
    </div>

    <n-modal v-model:show="showAdvanced"
             preset="dialog"
             title="高级配置"
             :show-icon="false"
             style="width: 500px"
    >
      <n-form label-width="85"
              label-placement="left"
              size="small">
        <n-grid :cols="2">
          <n-gi>
            <n-form-item label="调试模式">
              <n-checkbox v-model:checked="advancedOptions.debug"></n-checkbox>
            </n-form-item>
          </n-gi>

          <n-gi>
            <n-form-item label="禁用心跳包">
              <n-checkbox v-model:checked="advancedOptions.disable_heartbeat"></n-checkbox>
            </n-form-item>
          </n-gi>
        </n-grid>
        <n-grid :cols="2">
          <n-gi>
            <n-form-item label="超时时间(s)">
              <n-input-number v-model:value="advancedOptions.timeout"/>
            </n-form-item>
          </n-gi>
          <n-gi>
            <n-form-item label="缓冲区(B)">
              <n-input-number v-model:value="advancedOptions.buffer_size"/>
            </n-form-item>
          </n-gi>
        </n-grid>
        <n-form-item label="流量集中">
          <n-input v-model:value="advancedOptions.redirect_url"
                   placeholder="用于应对负载均衡，流量将集中转发到这个 url"/>
        </n-form-item>
        <n-form-item label="上游代理">
          <n-input v-model:value="advancedOptions.upstream_proxy" placeholder="socks5://user:pass@ip:port"/>
        </n-form-item>
        <n-form-item label="请求头">
          <n-input type="textarea" v-model:value="header"/>
        </n-form-item>
        <n-form-item label="地址">
          <n-a :href=link @click.stop.prevent="openLink">{{ link }}</n-a>
        </n-form-item>
      </n-form>
      <template #action>
        <n-button type="primary" @click="confirmAdvanced">确定</n-button>
      </template>
    </n-modal>
  </div>
</template>

<script lang="ts" setup>

import {ctrl, main} from "../wailsjs/go/models";
import {DefaultSuo5Config, RunSuo5WithConfig, Stop} from "../wailsjs/go/main/App";
import {BrowserOpenURL, EventsOn} from "../wailsjs/runtime";
import {AlertProps} from "naive-ui/es/alert/src/Alert";
import {ButtonProps, FormInst, useMessage} from 'naive-ui'
import {onBeforeMount} from "vue";
import Status = main.Status;

const message = useMessage()


const formValue = ref<ctrl.Suo5Config>({
  listen: '',
  target: '',
  no_auth: false,
  username: '',
  password: '',
  mode: '',
  buffer_size: 0,
  timeout: 0,
  debug: false,
  upstream_proxy: '',
  method: '',
  redirect_url: '',
  raw_header: [],
  disable_heartbeat: false,
})

const advancedOptions = ref<ctrl.Suo5Config>(Object.assign({}, formValue.value))

onBeforeMount(async () => {
  formValue.value = await DefaultSuo5Config();
  advancedOptions.value = await DefaultSuo5Config();
})

const header = computed({
  get() {
    return advancedOptions.value.raw_header.join('\n')
  },
  set(newValue) {
    advancedOptions.value.raw_header = newValue.split('\n')
  }
})

const showAdvanced = ref(false)
const confirmAdvanced = () => {
  formValue.value.debug = advancedOptions.value.debug
  formValue.value.timeout = advancedOptions.value.timeout
  formValue.value.buffer_size = advancedOptions.value.buffer_size
  formValue.value.upstream_proxy = advancedOptions.value.upstream_proxy
  formValue.value.raw_header = advancedOptions.value.raw_header
  formValue.value.redirect_url = advancedOptions.value.redirect_url
  formValue.value.disable_heartbeat = advancedOptions.value.disable_heartbeat
  showAdvanced.value = false
}
const formRef = ref<FormInst | null>(null)
const alertType = ref<AlertProps["type"]>("warning")
const alertContent = ref("还未连接")
const runLoading = ref(false);
const btnType = ref<ButtonProps["type"]>("primary")
const btnName = ref("连接");

const runAction = async () => {
  if (btnName.value == "连接") {
    if (formValue.value.target.trim() == "") {
      message.warning("请输入目标")
      return
    }
    checkingAction()
    await RunSuo5WithConfig(formValue.value)
  } else {
    await Stop()
    resetAction()
  }
}

interface ConnectedEvent {
  mode: string
}

const status = ref<Status>({
  connection_count: 0,
  memory_usage: "0MB",
  cpu_percent: "0%",
});

const log = ref('')
const logInst = ref()
const clearLog = () => {
  log.value = ''
}

onMounted(() => {
  watchEffect(() => {
    if (log.value) {
      nextTick(() => {
        logInst.value?.scrollTo({position: 'bottom', slient: true})
      })
    }
  })
})

onMounted(() => {
  EventsOn("connected", (e: ConnectedEvent) => {
    let mode = "全双工"
    if (e.mode == "half") {
      mode = "半双工"
    }
    let proxy = ""
    if (formValue.value.no_auth) {
      proxy = `socks5://${formValue.value.listen}`
    } else {
      proxy = `socks5://${formValue.value.username}:${formValue.value.password}@${formValue.value.listen}`
    }
    let msg = `连接成功, 当前工作在${mode}模式, 代理地址: ${proxy}`
    successAction(msg)
  })

  EventsOn("log", (e) => {
    log.value += e
  })

  EventsOn("error", (e) => {
    errorAction(e.toString())
  })

  EventsOn("status", (e: Status) => {
    status.value = e
  })
})
onUnmounted(Stop)


const resetAction = () => {
  runLoading.value = false
  btnType.value = "primary"
  btnName.value = "连接"
  alertType.value = "warning"
  alertContent.value = "还未连接"
}

const checkingAction = () => {
  log.value = ''
  runLoading.value = true;
  alertType.value = "info"
  alertContent.value = "正在连接..."
}

const successAction = (content: string) => {
  runLoading.value = false
  btnType.value = "success"
  btnName.value = "停止"
  alertType.value = "success"
  alertContent.value = content
}

const errorAction = (content: string) => {
  runLoading.value = false
  btnType.value = "primary"
  btnName.value = "连接"
  alertType.value = "warning"
  alertContent.value = content
}


const authChange = (enable: boolean) => {
  formValue.value.no_auth = !enable
  if (enable && formValue.value.username === "") {
    formValue.value.username = "suo5"
    formValue.value.password = randString(8)
  }
}

const randString = (length: number) => {
  const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
  let result = '';
  let charactersLength = characters.length;
  for (let i = 0; i < length; i++) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength));
  }
  return result;
}

const link = ref("https://github.com/zema1/suo5")
const openLink = () => {
  BrowserOpenURL(link.value)
}


</script>
<style lang="less" scoped>
@common-padding: 24px;

.container {
  padding: 12px @common-padding @common-padding @common-padding;
}

#param {
  width: 500px
}

#mode {
  align-items: center;
  width: 100%;
}

.footer {
  position: absolute;
  bottom: 1px;
}

.line-block {
  display: inline-block;
}

</style>