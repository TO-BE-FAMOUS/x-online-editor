<!-- @format -->

<template>
  <div class="index-page" v-loading="isLoading">
    <HeaderNav @toggle-sidebar="onToggleSidebar" />
    <div class="index-page__body">
      <Sidebar
        v-if="!isMobile"
        :collapsed="sidebarCollapsed"
        :active-doc-id="activeDocId"
        @select-doc="onSelectDoc"
        @doc-deleted="onDocDeleted"
        @toggle-sidebar="onToggleSidebar"
      />
      <div
        class="index-page__editor"
        :style="{ marginLeft: isMobile ? 0 : sidebarCollapsed ? '48px' : '260px' }"
      >
        <div
          v-if="unicodeOpen"
          class="ut-dropdown"
          :style="{ top: dropdownPos.top + 'px', left: dropdownPos.left + 'px' }"
          @mousedown.prevent
        >
          <div class="ut-dropdown-title">Unicode 样式（选中文本后点击）</div>
          <div class="ut-dropdown-grid">
            <button class="ut-btn" @mousedown.prevent="applyUnicode('bold')"><b>B</b> 粗体</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('italic')"><i>I</i> 斜体</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('underline')"><u>U</u> 下划线</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('strike')"><s>S</s> 删除线</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('mono')">Mono 等宽</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('script')">𝒮𝒸 手写</button>
            <button class="ut-btn" @mousedown.prevent="applyUnicode('clear')">Clear 清除</button>
          </div>
          <div class="ut-dropdown-hint">仅对英文/数字生效</div>
        </div>
        <div id="vditor" class="vditor" />
      </div>
    </div>
  </div>
</template>

<script>
import Vditor from 'vditor'
import 'vditor/src/assets/less/index.less'
import HeaderNav from './partials/HeaderNav'
import Sidebar from '@components/Sidebar'
import defaultText from '@config/default'
import emojiMap from '@config/emoji'
import {
  migrateFromLegacy,
  getDocuments,
  getActiveDocId,
  setActiveDocId,
  getDocContent,
  saveDocContent,
} from '@helper/storage'
import { trackEvent } from '@helper/analytics'

const SAVE_DEBOUNCE_MS = 1000

export default {
  name: 'index-page',

  data() {
    return {
      isLoading: true,
      isMobile: window.innerWidth <= 960,
      vditor: null,
      activeDocId: null,
      sidebarCollapsed: window.innerWidth <= 960,
      saveTimer: null,
      unicodeOpen: false,
      dropdownPos: { top: 0, left: 0 },
    }
  },

  created() {
    migrateFromLegacy(defaultText)
    if (getDocuments().length === 0) {
      const { createDocument } = require('@helper/storage')
      createDocument('未命名文档')
    }
    this.activeDocId = getActiveDocId() || (getDocuments()[0] && getDocuments()[0].id)
    if (this.activeDocId) {
      setActiveDocId(this.activeDocId)
    }
    console.log = () => {}
  },

  components: {
    HeaderNav,
    Sidebar,
  },

  mounted() {
    this.initVditor()
    this.$nextTick(() => {
      this.isLoading = false
    })
    this.$root.$on('reload-content', this.reloadContent)
    window.addEventListener('resize', this.onResize)
  },

  beforeDestroy() {
    this.saveCurrentDoc()
    this.$root.$off('reload-content', this.reloadContent)
    window.removeEventListener('resize', this.onResize)
    if (this.saveTimer) clearTimeout(this.saveTimer)
    if (this._unicodeOutsideHandler) {
      document.removeEventListener('mousedown', this._unicodeOutsideHandler)
    }
    if (this._paneObserver) this._paneObserver.disconnect()
  },

  methods: {
    onResize() {
      this.isMobile = window.innerWidth <= 960
    },
    onToggleSidebar() {
      this.sidebarCollapsed = !this.sidebarCollapsed
      trackEvent('sidebar_toggle', 'sidebar', this.sidebarCollapsed ? 'collapse' : 'expand')
    },
    initVditor() {
      const that = this
      const options = {
        width: '100%',
        height: window.innerHeight - 120,
        tab: '\t',
        counter: '999999',
        typewriterMode: false,
        mode: 'sv',
        cache: { enable: false },
        preview: {
          delay: 100,
          show: !this.isMobile,
        },
        resize: {
          enable: false,
        },
        hint: {
          emoji: emojiMap,
          emojiPath: '',
        },
        outline: true,
        upload: {
          max: 5 * 1024 * 1024,
          handler(file) {
            let formData = new FormData()
            for (let i in file) {
              formData.append('smfile', file[i])
            }
            let request = new XMLHttpRequest()
            request.open('POST', 'https://sm.ms/api/upload')
            request.onload = that.onloadCallback
            request.send(formData)
          },
        },
        input: (value) => {
          that.debouncedSave(value)
        },
        after: () => {
          const content = getDocContent(this.activeDocId) || defaultText
          this.vditor.setValue(content)
          this.vditor.focus()
          this.injectUnicodeButton()
          this.injectCopyButton()
          this.moveActionToToolbar()
          this.lockPaneWidths()
          this.setupScrollSync()
        },
      }
      this.vditor = new Vditor('vditor', options)
    },
    debouncedSave(value) {
      if (this.saveTimer) clearTimeout(this.saveTimer)
      this.saveTimer = setTimeout(() => {
        if (this.activeDocId && this.vditor && typeof this.vditor.getValue === 'function') {
          const content = this.vditor.getValue()
          saveDocContent(this.activeDocId, content)
        }
        this.saveTimer = null
      }, SAVE_DEBOUNCE_MS)
    },
    saveCurrentDoc() {
      if (this.saveTimer) {
        clearTimeout(this.saveTimer)
        this.saveTimer = null
      }
      if (this.activeDocId && this.vditor && typeof this.vditor.getValue === 'function') {
        saveDocContent(this.activeDocId, this.vditor.getValue())
      }
    },
    onSelectDoc(id) {
      this.saveCurrentDoc()
      setActiveDocId(id)
      this.activeDocId = id
      const content = getDocContent(id) || ''
      this.vditor.setValue(content)
      this.vditor.focus()
      if (this.isMobile) this.sidebarCollapsed = true
    },
    onDocDeleted() {
      this.activeDocId = getActiveDocId()
      if (this.activeDocId && this.vditor) {
        this.vditor.setValue(getDocContent(this.activeDocId) || '')
        this.vditor.focus()
      } else if (this.vditor) {
        this.vditor.setValue('')
      }
    },
    onloadCallback(oEvent) {
      const currentTarget = oEvent.currentTarget
      if (currentTarget.status !== 200) {
        trackEvent('editor_image_upload_error', 'editor', currentTarget.statusText)
        return this.$message({
          type: 'error',
          message: currentTarget.status + ' ' + currentTarget.statusText,
        })
      }
      let resp = JSON.parse(currentTarget.response)
      let imgMdStr = ''
      if (resp.code === 'invalid_source') {
        trackEvent('editor_image_upload_invalid', 'editor', resp.message)
        return this.$message({
          type: 'error',
          message: resp.message,
        })
      }
      if (resp.code === 'image_repeated') {
        imgMdStr = `![](${resp.images})`
      } else if (resp.code === 'success' || resp.success) {
        imgMdStr = `![${resp.data.filename}](${resp.data.url})`
      }
      this.vditor.insertValue(imgMdStr)
      trackEvent('editor_image_upload_success', 'editor', resp.data ? resp.data.filename : '')
    },
    setupScrollSync() {
      const trySetup = () => {
        const sv = document.querySelector('.vditor-sv, .vditor-ir, .vditor-wysiwyg')
        const preview = document.querySelector('.vditor-preview')
        if (!sv || !preview) return false
        let syncing = false
        const makeHandler = (src, dst) => () => {
          if (syncing) return
          syncing = true
          const srcMax = src.scrollHeight - src.clientHeight
          const dstMax = dst.scrollHeight - dst.clientHeight
          const ratio = srcMax > 0 ? src.scrollTop / srcMax : 0
          dst.scrollTop = ratio * dstMax
          requestAnimationFrame(() => { syncing = false })
        }
        sv.addEventListener('scroll', makeHandler(sv, preview))
        preview.addEventListener('scroll', makeHandler(preview, sv))
        return true
      }
      if (!trySetup()) setTimeout(trySetup, 500)
    },
    moveActionToToolbar() {
      const tryMove = () => {
        const toolbar = document.querySelector('.vditor-toolbar')
        const action = document.querySelector('.vditor-preview__action')
        if (toolbar && action && action.parentElement !== toolbar) {
          toolbar.appendChild(action)
          return true
        }
        return !!(toolbar && action && action.parentElement === toolbar)
      }
      if (!tryMove()) {
        setTimeout(tryMove, 400)
        setTimeout(tryMove, 1000)
      }
    },
    injectCopyButton() {
      const tryInject = () => {
        const action = document.querySelector('.vditor-preview__action')
        if (!action) return false
        if (action.querySelector('.preview-copy-btn')) return true
        const btn = document.createElement('button')
        btn.className = 'preview-copy-btn'
        btn.type = 'button'
        btn.setAttribute('aria-label', '复制 Markdown')
        btn.innerHTML = `<svg viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg"><path d="M16 1H4C2.9 1 2 1.9 2 3v14h2V3h12V1zm3 4H8C6.9 5 6 5.9 6 7v14c0 1.1.9 2 2 2h11c1.1 0 2-.9 2-2V7c0-1.1-.9-2-2-2zm0 16H8V7h11v14z"/></svg>`
        btn.addEventListener('click', async (e) => {
          e.preventDefault()
          e.stopPropagation()
          const text = this.vditor && this.vditor.getValue ? this.vditor.getValue() : ''
          try {
            await navigator.clipboard.writeText(text)
          } catch {
            const ta = document.createElement('textarea')
            ta.value = text
            document.body.appendChild(ta)
            ta.select()
            document.execCommand('copy')
            document.body.removeChild(ta)
          }
          const orig = btn.innerHTML
          btn.innerHTML = '<span style="font-size:12px;">✓</span>'
          setTimeout(() => { btn.innerHTML = orig }, 1400)
        })
        action.appendChild(btn)
        return true
      }
      if (!tryInject()) {
        setTimeout(tryInject, 300)
        setTimeout(tryInject, 800)
      }
    },
    lockPaneWidths() {
      const content = document.querySelector('.vditor-content')
      if (!content) return
      const applyLock = () => {
        const sv = content.querySelector('.vditor-sv, .vditor-ir, .vditor-wysiwyg')
        const preview = content.querySelector('.vditor-preview')
        if (sv) {
          sv.style.width = '50%'
          sv.style.maxWidth = '50%'
          sv.style.minWidth = '50%'
          sv.style.flex = '0 0 50%'
        }
        if (preview) {
          preview.style.width = '50%'
          preview.style.maxWidth = '50%'
          preview.style.minWidth = '50%'
          preview.style.flex = '0 0 50%'
          preview.style.display = 'block'
          preview.style.left = 'auto'
          preview.style.position = 'relative'
        }
      }
      applyLock()
      if (this._paneObserver) this._paneObserver.disconnect()
      this._paneObserver = new MutationObserver(() => applyLock())
      this._paneObserver.observe(content, {
        attributes: true,
        subtree: true,
        attributeFilter: ['style', 'class'],
        childList: true,
      })
    },
    injectUnicodeButton() {
      const toolbar = document.querySelector('.vditor-toolbar')
      if (!toolbar || toolbar.querySelector('.ut-toolbar-btn')) return
      const moreBtn = toolbar.querySelector('[data-type="more"]')
      const btn = document.createElement('div')
      btn.className = 'vditor-toolbar__item ut-toolbar-btn'
      btn.setAttribute('data-type', 'unicode')
      btn.innerHTML = `<button type="button" aria-label="Unicode 样式" class="vditor-tooltipped vditor-tooltipped__s" style="display:flex;align-items:center;gap:2px;">
        <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 7V4h16v3"/><path d="M9 20h6"/><path d="M12 4v16"/></svg>
        <span style="font-size:9px;">▾</span>
      </button>`
      btn.addEventListener('mousedown', (e) => {
        e.preventDefault()
        const rect = btn.getBoundingClientRect()
        this.dropdownPos = { top: rect.bottom + 6, left: rect.left }
        this.unicodeOpen = !this.unicodeOpen
      })
      if (moreBtn && moreBtn.nextSibling) {
        moreBtn.parentNode.insertBefore(btn, moreBtn.nextSibling)
      } else if (moreBtn) {
        moreBtn.parentNode.appendChild(btn)
      } else {
        toolbar.appendChild(btn)
      }
      // close on outside click
      if (!this._unicodeOutsideHandler) {
        this._unicodeOutsideHandler = (e) => {
          if (!this.unicodeOpen) return
          if (e.target.closest('.ut-dropdown') || e.target.closest('.ut-toolbar-btn')) return
          this.unicodeOpen = false
        }
        document.addEventListener('mousedown', this._unicodeOutsideHandler)
      }
    },
    applyUnicode(op) {
      const UNDERLINE = '\u0332'
      const STRIKE = '\u0336'
      const boldMap = {a:'𝗮',b:'𝗯',c:'𝗰',d:'𝗱',e:'𝗲',f:'𝗳',g:'𝗴',h:'𝗵',i:'𝗶',j:'𝗷',k:'𝗸',l:'𝗹',m:'𝗺',n:'𝗻',o:'𝗼',p:'𝗽',q:'𝗾',r:'𝗿',s:'𝘀',t:'𝘁',u:'𝘂',v:'𝘃',w:'𝘄',x:'𝘅',y:'𝘆',z:'𝘇',A:'𝗔',B:'𝗕',C:'𝗖',D:'𝗗',E:'𝗘',F:'𝗙',G:'𝗚',H:'𝗛',I:'𝗜',J:'𝗝',K:'𝗞',L:'𝗟',M:'𝗠',N:'𝗡',O:'𝗢',P:'𝗣',Q:'𝗤',R:'𝗥',S:'𝗦',T:'𝗧',U:'𝗨',V:'𝗩',W:'𝗪',X:'𝗫',Y:'𝗬',Z:'𝗭','0':'𝟬','1':'𝟭','2':'𝟮','3':'𝟯','4':'𝟰','5':'𝟱','6':'𝟲','7':'𝟳','8':'𝟴','9':'𝟵'}
      const italicMap = {a:'𝑎',b:'𝑏',c:'𝑐',d:'𝑑',e:'𝑒',f:'𝑓',g:'𝑔',h:'ℎ',i:'𝑖',j:'𝑗',k:'𝑘',l:'𝑙',m:'𝑚',n:'𝑛',o:'𝑜',p:'𝑝',q:'𝑞',r:'𝑟',s:'𝑠',t:'𝑡',u:'𝑢',v:'𝑣',w:'𝑤',x:'𝑥',y:'𝑦',z:'𝑧',A:'𝐴',B:'𝐵',C:'𝐶',D:'𝐷',E:'𝐸',F:'𝐹',G:'𝐺',H:'𝐻',I:'𝐼',J:'𝐽',K:'𝐾',L:'𝐿',M:'𝑀',N:'𝑁',O:'𝑂',P:'𝑃',Q:'𝑄',R:'𝑅',S:'𝑆',T:'𝑇',U:'𝑈',V:'𝑉',W:'𝑊',X:'𝑋',Y:'𝑌',Z:'𝑍'}
      const monoMap = {}
      {
        const lo='abcdefghijklmnopqrstuvwxyz', up='ABCDEFGHIJKLMNOPQRSTUVWXYZ', dg='0123456789'
        const mlo='𝚊𝚋𝚌𝚍𝚎𝚏𝚐𝚑𝚒𝚓𝚔𝚕𝚖𝚗𝚘𝚙𝚚𝚛𝚜𝚝𝚞𝚟𝚠𝚡𝚢𝚣'
        const mup='𝙰𝙱𝙲𝙳𝙴𝙵𝙶𝙷𝙸𝙹𝙺𝙻𝙼𝙽𝙾𝙿𝚀𝚁𝚂𝚃𝚄𝚅𝚆𝚇𝚈𝚉'
        const mdg='𝟶𝟷𝟸𝟹𝟺𝟻𝟼𝟽𝟾𝟿'
        ;[...lo].forEach((c,i) => (monoMap[c] = [...mlo][i]))
        ;[...up].forEach((c,i) => (monoMap[c] = [...mup][i]))
        ;[...dg].forEach((c,i) => (monoMap[c] = [...mdg][i]))
      }
      const scriptMap = {}
      {
        const lo='abcdefghijklmnopqrstuvwxyz', up='ABCDEFGHIJKLMNOPQRSTUVWXYZ'
        const slo='𝒶𝒷𝒸𝒹ℯ𝒻ℊ𝒽𝒾𝒿𝓀𝓁𝓂𝓃ℴ𝓅𝓆𝓇𝓈𝓉𝓊𝓋𝓌𝓍𝓎𝓏'
        const sup='𝒜ℬ𝒞𝒟ℰℱ𝒢ℋℐ𝒥𝒦ℒℳ𝒩𝒪𝒫𝒬ℛ𝒮𝒯𝒰𝒱𝒲𝒳𝒴𝒵'
        ;[...lo].forEach((c,i) => (scriptMap[c] = [...slo][i]))
        ;[...up].forEach((c,i) => (scriptMap[c] = [...sup][i]))
      }
      const reverseMaps = {}
      ;[boldMap, italicMap, monoMap, scriptMap].forEach((m) => {
        for (const k in m) reverseMaps[m[k]] = k
      })
      const toRegular = (s) => {
        let out = ''
        for (const ch of s) {
          if (ch === UNDERLINE || ch === STRIKE) continue
          out += reverseMaps[ch] || ch
        }
        return out
      }
      const mapTransform = (s, map) => {
        const base = toRegular(s)
        let out = ''
        for (const ch of base) out += map[ch] || ch
        return out
      }
      const transformers = {
        bold: (s) => mapTransform(s, boldMap),
        italic: (s) => mapTransform(s, italicMap),
        underline: (s) => {
          const t = s.split('').filter((c) => c !== UNDERLINE).join('')
          let out = ''
          for (const ch of t) out += ch + UNDERLINE
          return out
        },
        strike: (s) => {
          const t = s.split('').filter((c) => c !== STRIKE).join('')
          let out = ''
          for (const ch of t) out += ch + STRIKE
          return out
        },
        mono: (s) => mapTransform(s, monoMap),
        script: (s) => mapTransform(s, scriptMap),
        clear: (s) => toRegular(s),
      }
      const sel = window.getSelection()
      if (!sel || sel.rangeCount === 0 || sel.toString().length === 0) {
        this.$message && this.$message({ type: 'warning', message: '请先在编辑器中选中文本' })
        return
      }
      const selected = sel.toString()
      const replaced = transformers[op](selected)
      const ok = document.execCommand('insertText', false, replaced)
      if (!ok) {
        // fallback for textarea
        const ae = document.activeElement
        if (ae && (ae.tagName === 'TEXTAREA' || ae.tagName === 'INPUT')) {
          const start = ae.selectionStart
          const end = ae.selectionEnd
          ae.value = ae.value.slice(0, start) + replaced + ae.value.slice(end)
          ae.selectionStart = start
          ae.selectionEnd = start + replaced.length
          ae.dispatchEvent(new Event('input', { bubbles: true }))
        }
      }
    },
    reloadContent() {
      this.activeDocId = getActiveDocId()
      if (this.vditor && this.vditor.getValue) {
        const content = getDocContent(this.activeDocId) || ''
        this.vditor.setValue(content)
        this.vditor.focus()
      }
    },
  },
}
</script>

<style lang="less">
@import './../assets/styles/style.less';

.index-page {
  width: 100%;
  min-height: 100vh;
  height: 100%;
  background-color: @white;
  display: flex;
  flex-direction: column;

  .index-page__body {
    flex: 1;
    display: flex;
    margin-top: @header-height;
    margin-left: 0;
    margin-right: 0;
    width: 100%;
    max-width: none;
    min-height: calc(100vh - @header-height);
    overflow: hidden;
    position: relative;
  }

  .index-page__sidebar-overlay {
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    z-index: 99;
    background: rgba(0, 0, 0, 0.3);
    transition: opacity 0.2s ease;
  }

  .index-page__editor {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    padding: 1rem 0;
    max-width: none;
    margin: 0;
    width: 100%;
    transition: all 0.3s ease;
  }

  .ut-dropdown {
    position: fixed;
    background: #fff;
    border: 1px solid #cfd9de;
    border-radius: 8px;
    box-shadow: 0 4px 16px rgba(0, 0, 0, 0.15);
    padding: 12px;
    min-width: 260px;
    z-index: 10000;

    .ut-dropdown-title {
      font-size: 12px;
      color: #536471;
      margin-bottom: 8px;
    }

    .ut-dropdown-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 6px;
    }

    .ut-dropdown-hint {
      font-size: 11px;
      color: #8b98a5;
      margin-top: 8px;
    }

    .ut-btn {
      background: #fff;
      border: 1px solid #cfd9de;
      border-radius: 6px;
      padding: 6px 10px;
      font-size: 13px;
      cursor: pointer;
      color: #0f1419;
      text-align: left;
      transition: background 0.15s;

      &:hover { background: #eff3f4; }
      b { font-weight: 800; }
      i { font-style: italic; }
      u { text-decoration: underline; }
    }
  }
  // Preview action moved into toolbar area via JS
  .vditor-toolbar .vditor-preview__action {
    display: inline-flex !important;
    align-items: center;
    margin-left: auto;
    padding: 0 !important;
    background: transparent !important;
    border: none !important;
    text-align: right !important;
    position: static !important;
  }
  .vditor-toolbar {
    display: flex !important;
    align-items: center;
    flex-wrap: wrap;
  }
  .vditor-preview__action .preview-copy-btn {
    background-color: var(--toolbar-background-color, transparent);
    color: var(--toolbar-icon-color, #4c5158);
    line-height: 20px;
    border: 0;
    margin: 0 10px;
    cursor: pointer;
    padding: 0 7px;
    font-size: 12px;

    &:hover {
      color: var(--toolbar-icon-hover-color, #1e88e5);
    }
    &:focus { outline: none; }
    svg {
      fill: currentColor;
      height: 15px;
      width: 15px;
      vertical-align: middle;
    }
  }
  .ut-toolbar-btn {
    cursor: pointer;
    button { background: transparent; border: none; cursor: pointer; padding: 4px 6px; color: inherit; }
  }

  .vditor {
    width: 90% !important;
    max-width: 90% !important;
    margin: 0 auto;
    height: calc(100vh - @header-height - 48px);
    text-align: left;
    overflow: hidden;
    display: flex;
    flex-direction: column;

    .vditor-toolbar {
      border-bottom: none;
      background-color: #fcfcfc;
    }

    .vditor-content {
      height: auto;
      min-height: auto;
      border-top: none;
    }

    // Lock left/right pane widths (disable resizing)
    .vditor-content {
      display: flex !important;
      flex: 1;
      min-height: 0;
      overflow: hidden;
    }
    .vditor-sv,
    .vditor-preview {
      overflow-y: auto !important;
      height: 100% !important;
    }
    .vditor-sv,
    .vditor-ir,
    .vditor-wysiwyg {
      width: 50% !important;
      max-width: 50% !important;
      min-width: 50% !important;
      flex: 0 0 50% !important;
    }
    .vditor-preview {
      width: 50% !important;
      max-width: 50% !important;
      min-width: 50% !important;
      flex: 0 0 50% !important;
      left: auto !important;
      position: relative !important;
      display: block !important;
      visibility: visible !important;
    }
    .vditor-resize,
    .vditor-resize--right,
    .vditor-resize--top {
      display: none !important;
    }
  }

  .vditor-reset {
    font-size: 14px;
  }

  .vditor-textarea {
    font-size: 14px;
    height: 100% !important;
  }
}

@media (max-width: 960px) {
  .index-page {
    .index-page__editor {
      padding: 10px;
    }

    .vditor {
      min-height: calc(100vh - 60px - 20px);
    }
  }
}
</style>
