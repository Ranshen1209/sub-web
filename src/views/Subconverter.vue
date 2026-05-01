<template>
  <main class="subconverter-page">
    <section class="hero-section">
      <nav class="top-nav" aria-label="Primary navigation">
        <button class="brand-button" type="button" @click="goToProject">
          <span class="brand-mark">S</span>
          <span>Subconverter</span>
        </button>
        <div class="nav-meta">
          <span class="status-pill">{{ backendVersion || "Backend ready" }}</span>
          <button class="nav-link" type="button" @click="gotoGayhub">后端项目</button>
          <button class="nav-link" type="button" @click="gotoRemoteConfig">配置示例</button>
        </div>
      </nav>

      <div class="hero-grid">
        <div class="hero-copy">
          <h1>订阅短链生成</h1>
          <p class="hero-text">
            将订阅内容转换为便于分享和使用的短链接。超长内容会自动使用 Raw 镜像承载，保持最终链接简洁稳定。
          </p>
          <div class="hero-actions">
            <button class="cta-button" type="button" @click="makeUrlClick" :disabled="!canGenerateUrl">
              生成订阅链接
            </button>
            <button class="sand-button" type="button" @click="makeShortUrlClick" :disabled="!canGenerateShortUrl">
              {{ loading ? "生成中…" : "生成短链接" }}
            </button>
          </div>
        </div>

        <aside class="process-card" aria-label="处理流程">
          <ol>
            <li>读取订阅内容</li>
            <li>生成私有 Raw 镜像</li>
            <li>创建最终短链</li>
          </ol>
        </aside>
      </div>
    </section>

    <section class="workspace-section">
      <div class="workspace-card">
        <div class="section-heading">
          <h2>订阅转换</h2>
        </div>

        <el-form class="converter-form" :model="form" label-position="top">
          <div class="form-grid two-columns">
            <el-form-item label="模式设置">
              <div class="mode-segment" role="radiogroup" aria-label="模式设置">
                <button
                  ref="basicModeButton"
                  type="button"
                  class="mode-segment-button"
                  :class="{ 'is-active': advanced === '1' }"
                  role="radio"
                  :aria-checked="String(advanced === '1')"
                  :tabindex="advanced === '1' ? 0 : -1"
                  @click="setAdvancedMode('1')"
                  @keydown.left.prevent="setAdvancedMode('2')"
                  @keydown.up.prevent="setAdvancedMode('2')"
                  @keydown.right.prevent="setAdvancedMode('2')"
                  @keydown.down.prevent="setAdvancedMode('2')"
                >
                  基础模式
                </button>
                <button
                  ref="advancedModeButton"
                  type="button"
                  class="mode-segment-button"
                  :class="{ 'is-active': advanced === '2' }"
                  role="radio"
                  :aria-checked="String(advanced === '2')"
                  :tabindex="advanced === '2' ? 0 : -1"
                  @click="setAdvancedMode('2')"
                  @keydown.left.prevent="setAdvancedMode('1')"
                  @keydown.up.prevent="setAdvancedMode('1')"
                  @keydown.right.prevent="setAdvancedMode('1')"
                  @keydown.down.prevent="setAdvancedMode('1')"
                >
                  进阶模式
                </button>
              </div>
            </el-form-item>

            <el-form-item label="客户端">
              <el-select v-model="form.clientType" class="full-width warm-select" popper-class="warm-select-dropdown" aria-label="客户端">
                <el-option v-for="(v, k) in options.clientTypes" :key="k" :label="k" :value="v"></el-option>
              </el-select>
            </el-form-item>
          </div>

          <el-form-item label="订阅链接">
            <el-input
              v-model="form.sourceSubUrl"
              type="textarea"
              :rows="5"
              placeholder="支持订阅 URL 或 ss/ssr/vmess/trojan 等节点；多个链接可每行一个或用 | 分隔"
              aria-label="订阅链接"
              @blur="saveSubUrl"
            />
          </el-form-item>

          <div v-if="advanced === '2'" class="advanced-panel">
            <div class="section-heading compact">
              <h3>进阶参数</h3>
            </div>

            <el-form-item label="后端地址">
              <el-autocomplete
                v-model="form.customBackend"
                class="full-width"
                :fetch-suggestions="backendSearch"
                placeholder="例：http://127.0.0.1:25500/sub?"
                aria-label="后端地址"
              >
                <el-button slot="append" @click="gotoGayhub" icon="el-icon-link">项目</el-button>
              </el-autocomplete>
            </el-form-item>

            <el-form-item label="远程配置">
              <el-select v-model="form.remoteConfig" class="full-width warm-select" popper-class="warm-select-dropdown" allow-create filterable placeholder="请选择" aria-label="远程配置">
                <el-option-group v-for="group in options.remoteConfig" :key="group.label" :label="group.label">
                  <el-option v-for="item in group.options" :key="item.value" :label="item.label" :value="item.value"></el-option>
                </el-option-group>
              </el-select>
            </el-form-item>

            <div class="form-grid three-columns">
              <el-form-item label="Include">
                <el-input v-model="form.includeRemarks" placeholder="节点名包含关键字" aria-label="Include" />
              </el-form-item>
              <el-form-item label="Exclude">
                <el-input v-model="form.excludeRemarks" placeholder="节点名排除关键字" aria-label="Exclude" />
              </el-form-item>
              <el-form-item label="FileName">
                <el-input v-model="form.filename" placeholder="订阅文件名" aria-label="FileName" />
              </el-form-item>
            </div>

            <div class="custom-param-list">
              <el-form-item v-for="(param, i) in customParams" :key="i" class="custom-param-row">
                <el-input v-model="param.name" placeholder="自定义参数名" :aria-label="`第 ${i + 1} 个自定义参数名`" />
                <el-input v-model="param.value" placeholder="自定义参数内容" :aria-label="`第 ${i + 1} 个自定义参数内容`">
                  <el-button slot="suffix" class="icon-delete-button" type="text" icon="el-icon-delete" :aria-label="`删除第 ${i + 1} 个自定义参数`" :title="`删除第 ${i + 1} 个自定义参数`" @click="customParams.splice(i, 1)" />
                </el-input>
              </el-form-item>
            </div>

            <div class="option-strip">
              <el-checkbox v-model="form.nodeList" border>输出为 Node List</el-checkbox>
              <el-popover placement="bottom" v-model="form.extraset" :append-to-body="false">
                <div id="extra-options-panel" class="popover-options" role="region" aria-label="更多选项">
                  <el-checkbox v-model="form.emoji">Emoji</el-checkbox>
                  <el-checkbox v-model="form.scv">跳过证书验证</el-checkbox>
                  <el-checkbox v-model="form.udp" @change="needUdp = true">启用 UDP</el-checkbox>
                  <el-checkbox v-model="form.appendType">节点类型</el-checkbox>
                  <el-checkbox v-model="form.sort">排序节点</el-checkbox>
                  <el-checkbox v-model="form.fdn">过滤非法节点</el-checkbox>
                  <el-checkbox v-model="form.expand">规则展开</el-checkbox>
                </div>
                <el-button slot="reference" class="sand-el-button" aria-controls="extra-options-panel" :aria-expanded="String(form.extraset)">更多选项</el-button>
              </el-popover>
              <el-popover placement="bottom" v-model="featurePopoverVisible" :append-to-body="false">
                <div id="feature-options-panel" class="popover-options" role="region" aria-label="定制功能">
                  <el-checkbox v-model="form.tpl.surge.doh">Surge.DoH</el-checkbox>
                  <el-checkbox v-model="form.tpl.clash.doh">Clash.DoH</el-checkbox>
                  <el-checkbox v-model="form.insert">网易云</el-checkbox>
                </div>
                <el-button slot="reference" class="sand-el-button" aria-controls="feature-options-panel" :aria-expanded="String(featurePopoverVisible)">定制功能</el-button>
              </el-popover>
              <el-button class="sand-el-button" @click="addCustomParam" icon="el-icon-plus">添加参数</el-button>
            </div>
          </div>

          <div class="result-panel">
            <div class="section-heading compact">
              <h3>生成结果</h3>
            </div>
            <p class="sr-only" role="status" aria-live="polite">{{ statusMessage }}</p>
            <el-form-item label="定制订阅">
              <el-input class="copy-content" readonly v-model="customSubUrl" aria-label="定制订阅">
                <el-button slot="append" v-clipboard:copy="customSubUrl" v-clipboard:success="onCopy" icon="el-icon-document-copy" aria-label="复制定制订阅">复制</el-button>
              </el-input>
            </el-form-item>
            <el-form-item label="订阅短链">
              <el-input class="copy-content" readonly v-model="curtomShortSubUrl" aria-label="订阅短链">
                <el-button slot="append" v-clipboard:copy="curtomShortSubUrl" v-clipboard:success="onCopy" icon="el-icon-document-copy" aria-label="复制订阅短链">复制</el-button>
              </el-input>
            </el-form-item>
          </div>

          <div class="action-grid">
            <button class="cta-button" type="button" @click="makeUrlClick" :disabled="!canGenerateUrl">
              生成订阅链接
            </button>
            <button class="dark-button" type="button" @click="makeShortUrlClick" :disabled="!canGenerateShortUrl">
              {{ loading ? "生成中…" : "生成短链接" }}
            </button>
            <button class="sand-button" type="button" @click="dialogUploadConfigVisible = true" :disabled="loading">
              上传配置
            </button>
            <button class="sand-button" type="button" @click="clashInstall" :disabled="!canImportClash">
              一键导入 Clash
            </button>
            <button class="wide-sand-button" type="button" @click="dialogLoadConfigVisible = true" :disabled="loading">
              从 URL 解析
            </button>
          </div>
        </el-form>
      </div>
    </section>

    <ConfigUploadDialog
      :visible="dialogUploadConfigVisible"
      :upload-config="uploadConfig"
      :loading="loading"
      @cancel="handleUploadCancel"
      @confirm="handleConfigUpload"
    />

    <UrlParseDialog
      :visible="dialogLoadConfigVisible"
      :load-config="loadConfig"
      :loading="loading"
      @cancel="handleLoadCancel"
      @confirm="handleUrlParse"
    />
  </main>
</template>

<script>
// 导入配置
import { CONSTANTS } from '@/config/constants';
import { CLIENT_TYPES } from '@/config/client-types';
import { REMOTE_CONFIGS } from '@/config/remote-configs';

// 导入Composables
import { useSubscriptionForm, addCustomParam, saveSubUrl as saveSubscriptionUrl } from '@/composables/useSubscriptionForm';
import { useSubscription } from '@/composables/useSubscription';
import { useUrlParser } from '@/composables/useUrlParser';

// 导入工具函数
import { getLocalStorageItem } from '@/utils/storage';

const MAX_DIRECT_SHORT_URL_LENGTH = 8000;
const MAX_MIRROR_REMOTE_SOURCE_COUNT = 8;
const MAX_MIRROR_SOURCE_BYTES = 1 << 20;
const MIRROR_SOURCE_TIMEOUT_MS = 15000;
const SUBSCRIPTION_URL_PATTERN = /^https?:\/\//i;

// 导入服务
import { BackendService } from '@/services/backendService';
import { ShortUrlService } from '@/services/shortUrlService';
import { ConfigUploadService } from '@/services/configUploadService';

// 导入组件
import ConfigUploadDialog from '@/components/ConfigUploadDialog.vue';
import UrlParseDialog from '@/components/UrlParseDialog.vue';

export default {
  name: 'Subconverter',
  components: {
    ConfigUploadDialog,
    UrlParseDialog
  },
  data() {
    const subscriptionForm = useSubscriptionForm();

    return {
      // 配置选项
      options: {
        clientTypes: CLIENT_TYPES,
        backendOptions: [
          { value: CONSTANTS.DEFAULT_BACKEND },
          { value: "http://127.0.0.1:25500/sub?" }
        ],
        remoteConfig: REMOTE_CONFIGS
      },

      // 状态
      backendVersion: "",
      loading: false,
      curtomShortSubUrl: "",
      statusMessage: "",
      dialogUploadConfigVisible: false,
      loadConfig: "",
      dialogLoadConfigVisible: false,
      featurePopoverVisible: false,
      uploadConfig: "",
      subDocAdvanced: CONSTANTS.DOC_ADVANCED,

      // 是否为 PC 端
      isPC: true,

      // 合并表单状态
      ...subscriptionForm,
      form: {
        ...subscriptionForm.form,
        customBackend: CONSTANTS.DEFAULT_BACKEND
      }
    };
  },
  computed: {
    // 按钮统一样式
    buttonStyle() {
      return { width: '140px' };
    },

    canGenerateShortUrl() {
      return this.customSubUrl.length > 0 && !this.loading;
    },

    canGenerateUrl() {
      return this.form.sourceSubUrl.length > 0 && this.form.clientType;
    },

    canImportClash() {
      return this.customSubUrl.length > 0;
    },

    processedSubUrl() {
      return this.form.sourceSubUrl.replace(/(\n|\r|\n\r)/g, "|");
    },

    currentBackend() {
      return this.form.customBackend || CONSTANTS.DEFAULT_BACKEND;
    }
  },
  created() {
    document.title = "Subscription Converter";
    this.isPC = this.$getOS().isPc;

    // 获取 url cache
    if (import.meta.env.VITE_USE_STORAGE === 'true') {
      const cachedUrl = getLocalStorageItem('sourceSubUrl');
      if (cachedUrl) {
        this.form.sourceSubUrl = cachedUrl;
      }
    }
  },
  mounted() {
    this.form.clientType = CONSTANTS.DEFAULT_CLIENT_TYPE;
    this.getBackendVersion().catch(() => {
      this.backendVersion = "Backend status unavailable";
    });
    
    // 延迟加载隐私提示，避免阻塞页面初始化
    setTimeout(() => {
      this.notify();
    }, 1000);
  },
  methods: {
    onCopy() {
      this.statusMessage = "已复制到剪贴板";
      this.$message.success("Copied!");
    },

    setAdvancedMode(mode) {
      this.advanced = mode;
      this.$nextTick(() => {
        const refName = mode === '1' ? 'basicModeButton' : 'advancedModeButton';
        if (this.$refs[refName]) {
          this.$refs[refName].focus();
        }
      });
    },

    goToProject() {
      window.open(CONSTANTS.PROJECT);
    },

    gotoGayhub() {
      window.open(CONSTANTS.BACKEND_RELEASE);
    },

    gotoRemoteConfig() {
      window.open(CONSTANTS.REMOTE_CONFIG_SAMPLE);
    },

    clashInstall() {
      if (this.customSubUrl === "") {
        this.statusMessage = "请先填写必填项，生成订阅链接";
        this.$message.error("请先填写必填项，生成订阅链接");
        return false;
      }

      const url = "clash://install-config?url=";
      window.open(
        url +
        encodeURIComponent(
          this.curtomShortSubUrl !== ""
            ? this.curtomShortSubUrl
            : this.customSubUrl
        )
      );
    },

    makeUrlClick() {
      const url = this.makeUrl(this.form, this.advanced, this.processedSubUrl, this.currentBackend, this.customParams, this.needUdp);
      if (url) {
        this.customSubUrl = url;
        this.statusMessage = "定制订阅已生成并复制到剪贴板";
        this.$copyText(this.customSubUrl);
        this.$message.success("定制订阅已复制到剪贴板");
      } else {
        this.statusMessage = "订阅链接与客户端为必填项";
        this.$message.error("订阅链接与客户端为必填项");
      }
    },

    makeShortUrlClick() {
      if (this.customSubUrl === "") {
        this.statusMessage = "请先生成订阅链接，再获取对应短链接";
        this.$message.warning("请先生成订阅链接，再获取对应短链接");
        return false;
      }

      this.loading = true;

      this.buildShortUrlTarget()
        .then(targetUrl => ShortUrlService.generateShortUrl(this.$axios, targetUrl))
        .then(shortUrl => {
          this.curtomShortSubUrl = shortUrl;
          this.statusMessage = "短链接已生成并复制到剪贴板";
          this.$copyText(shortUrl);
          this.$message.success("短链接已复制到剪贴板");
        })
        .catch(error => {
          this.statusMessage = "短链接获取失败：" + error.message;
          this.$message.error("短链接获取失败：" + error.message);
        })
        .finally(() => {
          this.loading = false;
        });
    },

    async buildShortUrlTarget() {
      if (this.customSubUrl.length <= MAX_DIRECT_SHORT_URL_LENGTH) {
        return this.customSubUrl;
      }

      const url = new URL(this.customSubUrl);
      const originalSubUrl = url.searchParams.get("url");
      if (!originalSubUrl) {
        return this.customSubUrl;
      }

      const subscriptionContent = await this.resolveMirrorSubscriptionContent(originalSubUrl);
      const rawContent = this.encodeBase64(subscriptionContent.trim());
      const rawUrl = await ShortUrlService.generateRawSubscriptionUrl(this.$axios, rawContent);
      url.searchParams.set("url", rawUrl);
      return url.toString();
    },

    async resolveMirrorSubscriptionContent(source) {
      const sources = source
        .split(/[\n\r|]+/)
        .map(item => item.trim())
        .filter(Boolean);
      const remoteSources = sources.filter(item => SUBSCRIPTION_URL_PATTERN.test(item));
      if (remoteSources.length > MAX_MIRROR_REMOTE_SOURCE_COUNT) {
        throw new Error("远程订阅源数量过多，最多支持 " + MAX_MIRROR_REMOTE_SOURCE_COUNT + " 个");
      }

      const contents = await Promise.all(sources.map(item => this.resolveSingleSubscriptionSource(item)));
      return contents.map(content => this.toPlainSubscriptionContent(content)).join("\n");
    },

    async resolveSingleSubscriptionSource(source) {
      if (!SUBSCRIPTION_URL_PATTERN.test(source)) {
        return source;
      }
      if (this.isBlockedMirrorSource(source)) {
        throw new Error("不支持镜像同源、内网、本机或非 HTTPS 订阅地址");
      }
      if (this.requiresArbitraryHostnameConfirmation(source)) {
        await this.confirmTrustedSubscriptionHost(source);
      }

      if (typeof AbortController === "undefined") {
        throw new Error("当前浏览器不支持安全的请求超时控制，无法镜像远程订阅");
      }

      const controller = new AbortController();
      const timeout = window.setTimeout(() => controller.abort(), MIRROR_SOURCE_TIMEOUT_MS);
      try {
        const response = await fetch(source, {
          cache: "no-store",
          credentials: "omit",
          redirect: "error",
          signal: controller.signal
        });
        if (!response.ok) {
          throw new Error("浏览器获取订阅失败：HTTP " + response.status);
        }

        const contentLength = Number(response.headers.get("Content-Length") || 0);
        if (contentLength > MAX_MIRROR_SOURCE_BYTES) {
          throw new Error("订阅内容超过 1 MiB 限制");
        }

        return await this.readLimitedResponseText(response);
      } finally {
        window.clearTimeout(timeout);
      }
    },

    async readLimitedResponseText(response) {
      if (!response.body || !response.body.getReader) {
        throw new Error("当前浏览器不支持安全的流式读取，无法镜像远程订阅");
      }

      const reader = response.body.getReader();
      const decoder = new TextDecoder("utf-8");
      let byteLength = 0;
      let content = "";

      let isReading = true;
      while (isReading) {
        const result = await reader.read();
        if (result.done) {
          isReading = false;
        } else {
          byteLength += result.value.byteLength;
          if (byteLength > MAX_MIRROR_SOURCE_BYTES) {
            await reader.cancel();
            throw new Error("订阅内容超过 1 MiB 限制");
          }
          content += decoder.decode(result.value, { stream: true });
        }
      }

      return content + decoder.decode();
    },

    isBlockedMirrorSource(source) {
      const url = new URL(source);
      const hostname = this.normalizeMirrorHostname(url.hostname);
      if (url.protocol !== "https:") {
        return true;
      }
      if (url.origin === window.location.origin) {
        return true;
      }
      if (hostname === "localhost" || hostname.endsWith(".localhost") || hostname.endsWith(".local")) {
        return true;
      }
      return this.isBlockedIPv4Hostname(hostname) || this.isBlockedIPv6Hostname(hostname);
    },

    requiresArbitraryHostnameConfirmation(source) {
      const hostname = this.normalizeMirrorHostname(new URL(source).hostname);
      return !this.isIPAddressHostname(hostname);
    },

    async confirmTrustedSubscriptionHost(source) {
      const hostname = this.normalizeMirrorHostname(new URL(source).hostname);
      try {
        await this.$confirm(
          `即将读取 ${hostname} 的订阅内容。请只继续处理你信任的订阅源。`,
          "确认订阅来源",
          {
            confirmButtonText: "信任并继续",
            cancelButtonText: "取消",
            type: "warning",
            customClass: "trust-source-dialog"
          }
        );
      } catch (error) {
        throw new Error("已取消镜像外部订阅域名");
      }
    },

    isIPAddressHostname(hostname) {
      return /^(\d{1,3}\.){3}\d{1,3}$/.test(hostname) || hostname.includes(":");
    },

    normalizeMirrorHostname(hostname) {
      return hostname.toLowerCase().replace(/^\[|\]$/g, "").replace(/\.$/, "");
    },

    isBlockedIPv4Hostname(hostname) {
      if (!/^(\d{1,3}\.){3}\d{1,3}$/.test(hostname)) {
        return false;
      }

      const parts = hostname.split(".").map(value => Number(value));
      if (parts.some(value => !Number.isInteger(value) || value < 0 || value > 255)) {
        return true;
      }

      const [first, second] = parts;
      return first === 0 ||
        first === 10 ||
        first === 127 ||
        first >= 224 ||
        (first === 100 && second >= 64 && second <= 127) ||
        (first === 169 && second === 254) ||
        (first === 172 && second >= 16 && second <= 31) ||
        (first === 192 && second === 168);
    },

    isBlockedIPv6Hostname(hostname) {
      if (!hostname.includes(":")) {
        return false;
      }
      if (hostname === "::" || hostname === "::1" || hostname === "0:0:0:0:0:0:0:1") {
        return true;
      }
      const embeddedIPv4 = this.embeddedIPv4FromIPv6Hostname(hostname);
      if (embeddedIPv4) {
        return this.isBlockedIPv4Hostname(embeddedIPv4);
      }

      const firstHextet = parseInt(hostname.split(":").find(Boolean) || "0", 16);
      return firstHextet === 0 ||
        (firstHextet & 0xfe00) === 0xfc00 ||
        (firstHextet & 0xffc0) === 0xfe80 ||
        (firstHextet & 0xff00) === 0xff00;
    },

    embeddedIPv4FromIPv6Hostname(hostname) {
      if (hostname.includes(".")) {
        return hostname.slice(hostname.lastIndexOf(":") + 1);
      }

      const tail = hostname.startsWith("::ffff:")
        ? hostname.slice("::ffff:".length)
        : hostname.startsWith("::")
          ? hostname.slice("::".length)
          : "";
      const parts = tail.split(":");
      if (parts.length !== 2) {
        return "";
      }

      const numbers = parts.map(value => parseInt(value, 16));
      if (numbers.some(value => !Number.isInteger(value) || value < 0 || value > 0xffff)) {
        return "";
      }

      return [
        numbers[0] >> 8,
        numbers[0] & 0xff,
        numbers[1] >> 8,
        numbers[1] & 0xff
      ].join(".");
    },

    toPlainSubscriptionContent(content) {
      const normalizedContent = content.trim();
      const decodedContent = this.decodeBase64SubscriptionContent(normalizedContent);
      return decodedContent || normalizedContent;
    },

    decodeBase64SubscriptionContent(content) {
      try {
        const decodedContent = decodeURIComponent(escape(atob(content)));
        if (/(^|\n|\r)(ss|ssr|vmess|vless|trojan|hysteria2|hy2|tuic|https?):\/\//i.test(decodedContent)) {
          return decodedContent;
        }
      } catch (error) {
        return "";
      }
      return "";
    },

    encodeBase64(content) {
      return btoa(unescape(encodeURIComponent(content)));
    },

    confirmUploadConfig() {
      if (this.uploadConfig === "") {
        this.statusMessage = "远程配置不能为空";
        this.$message.warning("远程配置不能为空");
        return false;
      }

      this.loading = true;

      ConfigUploadService.uploadConfig(this.$axios, this.uploadConfig)
        .then(res => {
          const result = ConfigUploadService.handleUploadSuccess(res, this.$copyText, this.$message);
          if (result.success) {
            // 自动填充至『表单-远程配置』
            this.form.remoteConfig = result.url;
            this.statusMessage = "远程配置已上传并复制到剪贴板";
            this.$copyText(this.form.remoteConfig);
            this.dialogUploadConfigVisible = false;
            this.uploadConfig = "";
          }
        })
        .catch(error => {
          this.statusMessage = "远程配置上传失败: " + error.message;
          this.$message.error("远程配置上传失败: " + error.message);
        })
        .finally(() => {
          this.loading = false;
        });
    },

    handleUploadCancel() {
      this.uploadConfig = "";
      this.dialogUploadConfigVisible = false;
    },

    handleConfigUpload(configContent) {
      this.uploadConfig = configContent;
      this.confirmUploadConfig();
    },

    handleLoadCancel() {
      this.loadConfig = "";
      this.dialogLoadConfigVisible = false;
    },

    handleUrlParse(url) {
      this.loadConfig = url;
      this.confirmLoadConfig();
    },

    confirmLoadConfig() {
      this.loading = true;

      this.parseUrl(
        this.loadConfig,
        this.form,
        this.customParams,
        () => {
          this.dialogLoadConfigVisible = false;
          this.loadConfig = "";
          this.statusMessage = "长/短链接已成功解析为订阅信息";
          this.$message.success("长/短链接已成功解析为订阅信息");
        },
        (error) => {
          this.statusMessage = error;
          this.$message.error(error);
        }
      ).then(() => {
        this.loading = false;
      }).catch(() => {
        this.loading = false;
      });
    },

    backendSearch(queryString, cb) {
      const results = this.backendSearchSuggestions(queryString, this.options.backendOptions);
      cb(results);
    },

    backendSearchSuggestions(queryString, backends) {
      if (queryString) {
        return backends.filter(backend => {
          return backend.value.toLowerCase().indexOf(queryString.toLowerCase()) === 0;
        });
      }
      return backends;
    },

    async getBackendVersion() {
      this.backendVersion = await BackendService.getBackendVersion(this.$axios);
    },

    notify() {
      const h = this.$createElement;
      this.statusMessage = "隐私提示：各种订阅链接（短链接服务除外）生成纯前端实现，无隐私问题。默认提供后端转换服务，隐私担忧者请自行搭建后端服务。";

      this.$notify({
        title: "隐私提示",
        type: "warning",
        customClass: "privacy-notice",
        message: h(
          "p",
          { class: "privacy-notice-message" },
          "各种订阅链接（短链接服务除外）生成纯前端实现，无隐私问题。默认提供后端转换服务，隐私担忧者请自行搭建后端服务。"
        )
      });
    },

    // 表单相关方法
    saveSubUrl() {
      saveSubscriptionUrl(this.form);
    },

    addCustomParam() {
      addCustomParam(this.customParams);
    },

    // 使用 composables
    ...useSubscription(),
    ...useUrlParser()
  }
};
</script>

<style scoped>
.subconverter-page {
  min-height: 100vh;
  background: #f5f4ed;
  color: #141413;
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  margin: -1px;
  padding: 0;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  border: 0;
  white-space: nowrap;
}

.hero-section,
.workspace-section {
  max-width: 1200px;
  margin: 0 auto;
  padding: 32px 24px;
}

.top-nav {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 12px;
  background: rgba(250, 249, 245, 0.78);
  border: 1px solid #f0eee6;
  border-radius: 16px;
  box-shadow: 0 0 0 1px rgba(209, 207, 197, 0.46);
  backdrop-filter: blur(18px);
}

.brand-button,
.nav-link,
.cta-button,
.sand-button,
.dark-button,
.wide-sand-button {
  min-height: 44px;
  border: 0;
  border-radius: 12px;
  font: 500 15px/1 Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  cursor: pointer;
  transition: transform 160ms ease, box-shadow 160ms ease, background 160ms ease;
}

.brand-button {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 8px 14px 8px 10px;
  background: #ffffff;
  color: #141413;
  box-shadow: 0 0 0 1px #8b887d;
}

.brand-mark {
  display: grid;
  width: 28px;
  height: 28px;
  place-items: center;
  border-radius: 9px;
  background: #a84f34;
  color: #faf9f5;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 18px;
}

.nav-meta {
  display: flex;
  align-items: center;
  flex-wrap: wrap;
  justify-content: flex-end;
  gap: 8px;
}

.status-pill {
  padding: 8px 12px;
  border-radius: 999px;
  background: #e8e6dc;
  color: #30302e;
  font-size: 13px;
  box-shadow: 0 0 0 1px #8b887d;
}

.nav-link {
  padding: 0 12px;
  background: transparent;
  color: #4d4c48;
}

.hero-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.2fr) minmax(280px, 0.8fr);
  gap: 32px;
  align-items: stretch;
  padding: 92px 0 56px;
}

.hero-copy,
.process-card,
.workspace-card {
  background: #faf9f5;
  border: 1px solid #f0eee6;
  box-shadow: rgba(0, 0, 0, 0.05) 0 4px 24px;
}

.hero-copy {
  padding: 44px;
  border-radius: 32px;
}

.eyebrow,
.card-kicker {
  margin: 0 0 12px;
  color: #8f442d;
  font-size: 12px;
  font-weight: 500;
  letter-spacing: 0.12px;
  text-transform: uppercase;
}

h1,
h2,
h3 {
  margin: 0;
  color: #141413;
  font-family: Georgia, "Times New Roman", serif;
  font-weight: 500;
  letter-spacing: normal;
}

h1 {
  max-width: 760px;
  font-size: clamp(40px, 7vw, 64px);
  line-height: 1.1;
}

h2 {
  font-size: clamp(32px, 5vw, 52px);
  line-height: 1.2;
}

h3 {
  font-size: 25px;
  line-height: 1.2;
}

.hero-text,
.section-heading p {
  color: #5e5d59;
  font-size: 20px;
  line-height: 1.6;
}

.hero-text {
  max-width: 680px;
  margin: 24px 0 0;
}

.hero-actions,
.action-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-top: 28px;
}

.cta-button {
  padding: 0 18px;
  background: #a84f34;
  color: #faf9f5;
  box-shadow: 0 0 0 1px #a84f34;
}

.sand-button,
.wide-sand-button {
  padding: 0 16px;
  background: #e8e6dc;
  color: #30302e;
  box-shadow: 0 0 0 1px #8b887d;
}

.dark-button {
  padding: 0 16px;
  background: #30302e;
  color: #faf9f5;
  box-shadow: 0 0 0 1px #30302e;
}

.wide-sand-button {
  min-width: 220px;
}

.cta-button:hover,
.sand-button:hover,
.dark-button:hover,
.wide-sand-button:hover,
.brand-button:hover {
  transform: translateY(-1px);
}

.brand-button:focus-visible,
.nav-link:focus-visible,
.cta-button:focus-visible,
.sand-button:focus-visible,
.dark-button:focus-visible,
.wide-sand-button:focus-visible,
.mode-segment-button:focus-visible,
.converter-form :deep(.el-button:focus),
.converter-form :deep(.el-radio-button__orig-radio:focus + .el-radio-button__inner),
.converter-form :deep(.el-input__inner:focus),
.converter-form :deep(.el-textarea__inner:focus),
.converter-form :deep(.el-select .el-input__inner:focus),
.converter-form :deep(.el-checkbox__input.is-focus .el-checkbox__inner) {
  outline: 2px solid #0b5cad;
  outline-offset: 2px;
  box-shadow: 0 0 0 4px #faf9f5, 0 0 0 6px #0b5cad;
}

button:disabled {
  cursor: not-allowed;
  opacity: 0.55;
  transform: none;
}

.process-card {
  display: flex;
  flex-direction: column;
  justify-content: center;
  min-height: 320px;
  padding: 32px;
  border-radius: 24px;
}

.process-card ol {
  display: grid;
  gap: 18px;
  margin: 0;
  padding: 0;
  list-style: none;
  counter-reset: flow;
}

.process-card li {
  position: relative;
  padding: 18px 18px 18px 58px;
  border-radius: 16px;
  background: #f5f4ed;
  color: #4d4c48;
  line-height: 1.5;
  box-shadow: inset 0 0 0 1px #8b887d;
  counter-increment: flow;
}

.process-card li::before {
  position: absolute;
  top: 16px;
  left: 16px;
  display: grid;
  width: 28px;
  height: 28px;
  place-items: center;
  border-radius: 50%;
  background: #30302e;
  color: #faf9f5;
  content: counter(flow);
  font-size: 13px;
}

.workspace-card {
  padding: 32px;
  border-radius: 24px;
}

.section-heading {
  max-width: 760px;
  margin-bottom: 28px;
}

.section-heading.compact {
  margin-bottom: 18px;
}

.section-heading.compact p {
  font-size: 15px;
}

.converter-form :deep(.el-form-item) {
  margin-bottom: 22px;
}

.converter-form :deep(.el-form-item__content) {
  line-height: 1.4;
}

.converter-form :deep(.el-form-item__label) {
  color: #4d4c48;
  font-weight: 500;
  line-height: 1.4;
}

.mode-segment {
  display: inline-flex;
  gap: 4px;
  padding: 4px;
  border: 1px solid #8b887d;
  border-radius: 16px;
  background: #f5f4ed;
}

.mode-segment-button {
  min-height: 44px;
  padding: 0 18px;
  border: 0;
  border-radius: 12px;
  background: transparent;
  color: #4d4c48;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
}

.mode-segment-button.is-active {
  background: #a84f34;
  color: #faf9f5;
  box-shadow: 0 6px 18px rgba(168, 79, 52, 0.24);
}

.converter-form :deep(.el-input__inner),
.converter-form :deep(.el-textarea__inner) {
  min-height: 44px;
  border-color: #8b887d;
  border-radius: 12px;
  background: #ffffff;
  color: #141413;
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  box-shadow: 0 0 0 1px rgba(209, 207, 197, 0.18);
}

.converter-form :deep(.el-textarea__inner) {
  padding: 14px 16px;
  line-height: 1.6;
  scrollbar-width: thin;
  scrollbar-color: #a84f34 #f0eee6;
}

.converter-form :deep(.el-textarea__inner::-webkit-scrollbar) {
  width: 10px;
}

.converter-form :deep(.el-textarea__inner::-webkit-scrollbar-track) {
  border-radius: 999px;
  background: #f0eee6;
}

.converter-form :deep(.el-textarea__inner::-webkit-scrollbar-thumb) {
  border: 2px solid #f0eee6;
  border-radius: 999px;
  background: #a84f34;
}

.converter-form :deep(.el-input__inner:focus),
.converter-form :deep(.el-textarea__inner:focus) {
  border-color: #0b5cad;
}

.converter-form :deep(.el-input-group__append) {
  width: 1%;
  padding: 0;
  border-color: #8b887d;
  border-left: 0;
  border-radius: 0 12px 12px 0;
  background: #e8e6dc;
  overflow: hidden;
}

.converter-form :deep(.el-input-group__append .el-button) {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 96px;
  min-height: 44px;
  margin: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
  color: #30302e;
  font-weight: 600;
}

.converter-form :deep(.el-input-group--append .el-input__inner) {
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
}

.copy-content :deep(.el-input__inner) {
  overflow: hidden;
  text-overflow: ellipsis;
}

.full-width {
  width: 100%;
}

.form-grid {
  display: grid;
  gap: 18px;
}

.two-columns {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.three-columns {
  grid-template-columns: repeat(3, minmax(0, 1fr));
}

.advanced-panel,
.result-panel {
  margin-top: 28px;
  padding: 24px;
  border: 1px solid #8b887d;
  border-radius: 20px;
  background: #f5f4ed;
}

.custom-param-row :deep(.el-form-item__content) {
  display: grid;
  grid-template-columns: minmax(160px, 0.4fr) minmax(0, 1fr);
  gap: 12px;
}

.custom-param-row :deep(.icon-delete-button) {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-width: 32px;
  min-height: 32px;
  padding: 0;
}

.option-strip {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 12px;
  margin-top: 8px;
}

.popover-options {
  display: grid;
  gap: 10px;
}

.sand-el-button,
.converter-form :deep(.el-button) {
  border-color: #8b887d;
  border-radius: 10px;
  background: #e8e6dc;
  color: #30302e;
}

.warm-select :deep(.el-input__inner) {
  cursor: pointer;
}

.converter-form :deep(.el-checkbox.is-bordered) {
  display: inline-flex;
  align-items: center;
  height: 44px;
  padding: 0 16px;
  border-color: #8b887d;
  border-radius: 14px;
  background: #faf9f5;
}

.converter-form :deep(.el-checkbox.is-bordered.is-checked) {
  border-color: #a84f34;
  background: #eee4dc;
}

.converter-form :deep(.el-checkbox__inner) {
  border-color: #8b887d;
  border-radius: 5px;
}

.converter-form :deep(.el-checkbox__input.is-checked .el-checkbox__inner) {
  border-color: #a84f34;
  background: #a84f34;
}

.converter-form :deep(.el-checkbox__label) {
  color: #4d4c48;
  font-weight: 600;
}

.converter-form :deep(.el-checkbox__input.is-checked + .el-checkbox__label) {
  color: #30302e;
}

.converter-form :deep(.el-radio-button__inner) {
  border-color: #8b887d;
  background: #faf9f5;
  color: #4d4c48;
}

.converter-form :deep(.el-radio-button__orig-radio:checked + .el-radio-button__inner) {
  border-color: #a84f34;
  background: #a84f34;
  color: #faf9f5;
  box-shadow: -1px 0 0 0 #a84f34;
}

.action-grid {
  align-items: center;
}

@media (max-width: 900px) {
  .hero-grid,
  .two-columns,
  .three-columns {
    grid-template-columns: 1fr;
  }

  .hero-copy,
  .workspace-card,
  .process-card {
    padding: 24px;
  }
}

@media (max-width: 640px) {
  .hero-section,
  .workspace-section {
    padding: 20px 14px;
  }

  .top-nav,
  .nav-meta,
  .hero-actions,
  .action-grid {
    align-items: stretch;
    flex-direction: column;
  }

  .brand-button,
  .nav-link,
  .cta-button,
  .sand-button,
  .dark-button,
  .wide-sand-button {
    width: 100%;
  }

  .custom-param-row :deep(.el-form-item__content) {
    grid-template-columns: 1fr;
  }
}
</style>

<style>
html {
  scrollbar-width: thin;
  scrollbar-color: #a84f34 #f0eee6;
}

body::-webkit-scrollbar,
.warm-select-dropdown .el-scrollbar__wrap::-webkit-scrollbar {
  width: 10px;
}

body::-webkit-scrollbar-track,
.warm-select-dropdown .el-scrollbar__wrap::-webkit-scrollbar-track {
  border-radius: 999px;
  background: #f0eee6;
}

body::-webkit-scrollbar-thumb,
.warm-select-dropdown .el-scrollbar__wrap::-webkit-scrollbar-thumb {
  border: 2px solid #f0eee6;
  border-radius: 999px;
  background: #a84f34;
}

.warm-select-dropdown {
  margin-top: 8px !important;
  border: 1px solid #8b887d;
  border-radius: 16px;
  background: #faf9f5;
  box-shadow: 0 18px 48px rgba(20, 20, 19, 0.14);
  overflow: hidden;
}

.warm-select-dropdown .el-select-dropdown__wrap {
  max-height: 320px;
}

.warm-select-dropdown .el-select-dropdown__item {
  height: 44px;
  margin: 4px 8px;
  padding: 0 14px;
  border-radius: 12px;
  color: #30302e;
  font-size: 15px;
  line-height: 44px;
}

.warm-select-dropdown .el-select-dropdown__item.hover,
.warm-select-dropdown .el-select-dropdown__item:hover {
  background: #f0eee6;
}

.warm-select-dropdown .el-select-dropdown__item.selected {
  background: #eee4dc;
  color: #8f442d;
  font-weight: 700;
  box-shadow: inset 0 0 0 1px rgba(168, 79, 52, 0.28);
}

.warm-select-dropdown .el-select-group__title {
  height: auto;
  padding: 14px 18px 8px;
  color: #6f6c63;
  font-size: 13px;
  font-weight: 700;
  line-height: 1.2;
}

.warm-select-dropdown .popper__arrow {
  display: none;
}

.trust-source-dialog {
  border-radius: 18px;
  background: #faf9f5;
  color: #141413;
}

.trust-source-dialog .el-message-box__title {
  color: #141413;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 22px;
  font-weight: 500;
}

.trust-source-dialog .el-message-box__content {
  color: #4d4c48;
  line-height: 1.6;
}

.trust-source-dialog .el-button--primary {
  border-color: #a84f34;
  background: #a84f34;
}

.privacy-notice {
  width: min(440px, calc(100vw - 28px));
  padding: 20px 22px 20px 20px;
  border: 1px solid #f0eee6;
  border-radius: 18px;
  background: #faf9f5;
  box-shadow: 0 18px 48px rgba(20, 20, 19, 0.14), inset 0 0 0 1px rgba(139, 136, 125, 0.18);
}

.privacy-notice .el-notification__icon {
  display: grid;
  width: 40px;
  height: 40px;
  place-items: center;
  border-radius: 13px;
  background: #a84f34;
  color: #faf9f5;
  font-size: 22px;
}

.privacy-notice .el-notification__group {
  min-width: 0;
  margin-left: 14px;
  margin-right: 18px;
}

.privacy-notice .el-notification__title {
  color: #141413;
  font-family: Georgia, "Times New Roman", serif;
  font-size: 24px;
  font-weight: 500;
  line-height: 1.2;
}

.privacy-notice .el-notification__content {
  margin-top: 10px;
}

.privacy-notice-message {
  margin: 0;
  color: #4d4c48;
  font-size: 16px;
  font-style: normal;
  line-height: 1.65;
}

.privacy-notice .el-notification__closeBtn {
  top: 18px;
  right: 18px;
  color: #6f6c63;
  font-size: 18px;
}

.privacy-notice .el-notification__closeBtn:hover {
  color: #141413;
}

@media (max-width: 640px) {
  .privacy-notice {
    right: 14px !important;
    padding: 18px;
  }

  .privacy-notice .el-notification__title {
    font-size: 21px;
  }

  .privacy-notice-message {
    font-size: 15px;
  }
}
</style>
