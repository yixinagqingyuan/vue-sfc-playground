<script setup lang="ts">
import Message from '../Message.vue'
import {
  type Ref,
  type WatchStopHandle,
  inject,
  onMounted,
  onUnmounted,
  ref,
  watch,
  watchEffect,
} from 'vue'
// 引入 iframe 链接
import srcdoc from './srcdoc.html?raw'
import { PreviewProxy } from './PreviewProxy'
import { compileModulesForPreview } from './moduleCompiler'
import type { Props } from '../Repl.vue'
import { injectKeyStore } from '../../src/types'

const props = defineProps<{ show: boolean; ssr: boolean }>()

const store = inject(injectKeyStore)!
const clearConsole = inject<Ref<boolean>>('clear-console')!
const theme = inject<Ref<'dark' | 'light'>>('theme')!
const previewTheme = inject<Ref<boolean>>('preview-theme')!

const previewOptions = inject<Props['previewOptions']>('preview-options')

const container = ref()
const runtimeError = ref()
const runtimeWarning = ref()

let sandbox: HTMLIFrameElement
let proxy: PreviewProxy
let stopUpdateWatcher: WatchStopHandle | undefined

// create sandbox on mount
onMounted(createSandbox)

// reset sandbox when import map changes
watch(
  () => store.getImportMap(),
  () => {
    try {
      createSandbox()
    } catch (e: any) {
      store.errors = [e as Error]
      return
    }
  },
)

function switchPreviewTheme() {
  if (!previewTheme.value) return

  const html = sandbox.contentDocument?.documentElement

  if (html) {
    html.className = theme.value
  } else {
    // re-create sandbox
    createSandbox()
  }
}

// reset theme
watch([theme, previewTheme], switchPreviewTheme)

onUnmounted(() => {
  proxy.destroy()
  stopUpdateWatcher && stopUpdateWatcher()
})

//创建沙箱
function createSandbox() {
  // 如果有，就清除事和实例
  if (sandbox) {
    // clear prev sandbox
    proxy.destroy()
    stopUpdateWatcher && stopUpdateWatcher()
    container.value.removeChild(sandbox)
  }
  // 初始化 iframe
  sandbox = document.createElement('iframe')
  // 添加必要属性

  // 1. sandbox=""
  // 　　应用所有限制

  // 2. sandbox="allow-same-origin"
  // 　　允许 iframe 内容被视为与包含文档有相同的来源。

  // 3. sandbox="allow-top-navigation"
  // 　　允许 iframe 内容从包含文档导航（加载）内容。
  // 　　可用于禁用外部网站的JS跳转、target="_parent"、target="_top"等

  // 4. sandbox="allow-forms"
  // 　　允许表单提交。

  // 5. sandbox="allow-scripts"
  // 　　允许脚本执行，即允许iframe运行脚本（但不创建弹出窗口）。
  // 　　可用于禁用外部网站的JS

  // 6. sandbox="allow-popups"
  // 　　允许弹出窗口（如window.open，target="_blank"）。

  // 5. sandbox="allow-scripts"
  // 　　允许弹出窗口逃离沙箱：允许一个沙箱文件打开新窗口不强制使用沙盒。
  sandbox.setAttribute(
    'sandbox',
    [
      'allow-forms',
      'allow-modals',
      'allow-pointer-lock',
      'allow-popups',
      'allow-same-origin',
      'allow-scripts',
      'allow-top-navigation-by-user-activation',
    ].join(' '),
  )
  // 引入文件vue,和引入 vue-ssr
  const importMap = store.getImportMap()
  console.log(importMap)
  // 替换主题等，内容
  const sandboxSrc = srcdoc
    .replace(
      /<html>/,
      `<html class="${previewTheme.value ? theme.value : ''}">`,
    )
    .replace(/<!--IMPORT_MAP-->/, JSON.stringify(importMap)) // 使用importMap模式导入vue 以及vuesss的包
    .replace(
      /<!-- PREVIEW-OPTIONS-HEAD-HTML -->/,
      previewOptions?.headHTML || '', // 占位符head
    )
    .replace(
      /<!--PREVIEW-OPTIONS-PLACEHOLDER-HTML-->/, // 占位符内容
      previewOptions?.placeholderHTML || '',
    )
  //赋值 iframe-container
  sandbox.srcdoc = sandboxSrc
  //挂在 iframe
  container.value.appendChild(sandbox)
  /// 初始化沙箱，建立通信
  proxy = new PreviewProxy(sandbox, {
    on_fetch_progress: (progress: any) => {
      // pending_imports = progress;
    },
    // 执行错误回调
    on_error: (event: any) => {
      const msg =
        event.value instanceof Error ? event.value.message : event.value
      if (
        msg.includes('Failed to resolve module specifier') ||
        msg.includes('Error resolving module specifier')
      ) {
        runtimeError.value =
          msg.replace(/\. Relative references must.*$/, '') +
          `.\nTip: edit the "Import Map" tab to specify import paths for dependencies.`
      } else {
        runtimeError.value = event.value
      }
    },
    on_unhandled_rejection: (event: any) => {
      let error = event.value
      if (typeof error === 'string') {
        error = { message: error }
      }
      runtimeError.value = 'Uncaught (in promise): ' + error.message
    },
    // console的回调事件
    on_console: (log: any) => {
      // console的回调事件
      if (log.duplicate) {
        return
      }
      if (log.level === 'error') {
        if (log.args[0] instanceof Error) {
          runtimeError.value = log.args[0].message
        } else {
          runtimeError.value = log.args[0]
        }
      } else if (log.level === 'warn') {
        if (log.args[0].toString().includes('[Vue warn]')) {
          runtimeWarning.value = log.args
            .join('')
            .replace(/\[Vue warn\]:/, '')
            .trim()
        }
      }
    },
    on_console_group: (action: any) => {
      // group_logs(action.label, false);
    },
    on_console_group_end: () => {
      // ungroup_logs();
    },
    on_console_group_collapsed: (action: any) => {
      // group_logs(action.label, true);
    },
  })
  //iframe 加载完毕
  sandbox.addEventListener('load', () => {
    // 为a标签啥的绑定时事件，并且阻止a标签的默认行为，防止页面内打开跳转
    proxy.handle_links()
    // 监听变动
    stopUpdateWatcher = watchEffect(updatePreview)
    // 设置主题
    switchPreviewTheme()
  })
}

// 主要的编译逻辑
async function updatePreview() {
  // 清楚控制台
  if (import.meta.env.PROD && clearConsole.value) {
    console.clear()
  }
  runtimeError.value = null
  runtimeWarning.value = null

  let isSSR = props.ssr
  if (store.vueVersion) {
    const [major, minor, patch] = store.vueVersion
      .split('.')
      .map((v) => parseInt(v, 10))
    if (major === 3 && (minor < 2 || (minor === 2 && patch < 27))) {
      alert(
        `The selected version of Vue (${store.vueVersion}) does not support in-browser SSR.` +
          ` Rendering in client mode instead.`,
      )
      isSSR = false
    }
  }

  try {
    const mainFile = store.mainFile
    // 可以直接输出html
    // if SSR, generate the SSR bundle and eval it to render the HTML
    if (isSSR && mainFile.endsWith('.vue')) {
      const ssrModules = compileModulesForPreview(store, true)
      //debugger
      console.info(
        `[@vue/repl] successfully compiled ${ssrModules.length} modules for SSR.`,
      )
      //debugger
      await proxy.eval([
        `const __modules__ = {};`,
        ...ssrModules,
        `import { renderToString as _renderToString } from 'vue/server-renderer'
         import { createSSRApp as _createApp } from 'vue'
         const AppComponent = __modules__["${mainFile}"].default
         AppComponent.name = 'Repl'
         const app = _createApp(AppComponent)
         if (!app.config.hasOwnProperty('unwrapInjectedRef')) {
           app.config.unwrapInjectedRef = true
         }
         app.config.warnHandler = () => {}
         window.__ssr_promise__ = _renderToString(app).then(html => {
           document.body.innerHTML = '<div id="app">' + html + '</div>' + \`${
             previewOptions?.bodyHTML || ''
           }\`
         }).catch(err => {
           console.error("SSR Error", err)
         })
        `,
      ])
    }

    // compile code to simulated module system
    // 编译代码
    const modules = compileModulesForPreview(store)
    console.info(
      `[@vue/repl] successfully compiled ${modules.length} module${
        modules.length > 1 ? `s` : ``
      }.`,
    )

    const codeToEval = [
      `window.__modules__ = {};window.__css__ = [];` +
        `if (window.__app__) window.__app__.unmount();` +
        (isSSR
          ? ``
          : `document.body.innerHTML = '<div id="app"></div>' + \`${
              previewOptions?.bodyHTML || ''
            }\``),
      ...modules,
      `document.querySelectorAll('style[css]').forEach(el => el.remove())
        document.head.insertAdjacentHTML('beforeend', window.__css__.map(s => \`<style css>\${s}</style>\`).join('\\n'))`,
    ]

    // if main file is a vue file, mount it.
    if (mainFile.endsWith('.vue')) {
      codeToEval.push(
        `import { ${
          isSSR ? `createSSRApp` : `createApp`
        } as _createApp } from "vue"
        ${previewOptions?.customCode?.importCode || ''}
        const _mount = () => {
          const AppComponent = __modules__["${mainFile}"].default
          AppComponent.name = 'Repl'
          const app = window.__app__ = _createApp(AppComponent)
          if (!app.config.hasOwnProperty('unwrapInjectedRef')) {
            app.config.unwrapInjectedRef = true
          }
          app.config.errorHandler = e => console.error(e)
          ${previewOptions?.customCode?.useCode || ''}
          app.mount('#app')
        }
        if (window.__ssr_promise__) {
          window.__ssr_promise__.then(_mount)
        } else {
          _mount()
        }`,
      )
    }

    // eval code in sandbox
    await proxy.eval(codeToEval)
  } catch (e: any) {
    console.error(e)
    runtimeError.value = (e as Error).message
  }
}

/**
 * Reload the preview iframe
 */
function reload() {
  sandbox.contentWindow?.location.reload()
}

defineExpose({ reload })
</script>

<template>
  <div
    v-show="show"
    ref="container"
    class="iframe-container"
    :class="{ [theme]: previewTheme }"
  />
  <Message :err="runtimeError" />
  <Message v-if="!runtimeError" :warn="runtimeWarning" />
</template>

<style scoped>
.iframe-container,
.iframe-container :deep(iframe) {
  width: 100%;
  height: 100%;
  border: none;
  background-color: #fff;
}
.iframe-container.dark :deep(iframe) {
  background-color: #1e1e1e;
}
</style>
