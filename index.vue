<script>
import * as monaco from 'monaco-editor'
function noop() { }
export default {
  name: 'MonacoEditor',
  props: {
    diffEditor: { // 是否使用diff模式 ，默认是不支持
      type: Boolean,
      default: false
    },
    width: {
      type: [String, Number],
      default: '100%'
    },
    height: {
      type: [String, Number],
      default: '100%'
    },
    original: { // 只有在diff模式下有效
      type: [String, Object],
      default: ''
    },
    value: {
      type: [String, Object],
      default: ''
    },
    language: {
      type: String,
      default: 'json'
    },
    theme: {
      type: String,
      default: 'vs'
    },
    options: {
      type: Object,
      default() { return {} }
    },
    editorMounted: {
      type: Function,
      default: noop
    },
    editorBeforeMount: {
      type: Function,
      default: noop
    }
  },
  computed: {
    style() {
      return {
        width: !/^\d+$/.test(this.width) ? this.width : `${this.width}px`,
        height: !/^\d+$/.test(this.height) ? this.height : `${this.height}px`
      }
    }
  },
  watch: {
    options: {
      deep: true,
      handler(options) {
        this.editor && this.editor.updateOptions(options)
      }
    },
    value() {
      if (this.diffEditor) {
        return
      }
      this.editor && this.value !== this._getValue(this.editor) && this.editor.setValue(this.value)
    },
    language() {
      if (!this.editor) return
      if (this.diffEditor) { // diff模式下更新language
        const { original, modified } = this.editor.getModel()
        monaco.editor.setModelLanguage(original, this.language)
        monaco.editor.setModelLanguage(modified, this.language)
      } else {
        monaco.editor.setModelLanguage(this.editor.getModel(), this.language)
      }
    },
    theme() {
      this.editor && monaco.editor.setTheme(this.theme)
    },
    style() {
      this.editor && this.$nextTick(() => {
        this.editor.layout()
      })
    }
  },
  mounted() {
    this.initMonaco()
  },
  beforeDestroy() {
    this.editor && this.editor.dispose()
  },
  methods: {
    initMonaco() {
      const { value, language, theme, options } = this
      Object.assign(options, this._editorBeforeMount()) // 编辑器初始化前
      this.editor = monaco.editor[this.diffEditor ? 'createDiffEditor' : 'create'](this.$el, {
        value,
        language,
        theme,
        ...options
      })
      this.diffEditor && this._setModel(this.value, this.original)
      this._editorMounted(this.editor) // 编辑器初始化后
    },
    _setModel(value, original) { // diff模式下设置model
      const { language } = this
      const originalModel = monaco.editor.createModel(original, language)
      const modifiedModel = monaco.editor.createModel(value, language)
      this.editor.setModel({
        original: originalModel,
        modified: modifiedModel
      })
    },
    _editorBeforeMount() {
      const options = this.editorBeforeMount(monaco)
      return options || {}
    },
    _editorMounted(editor) {
      this.editorMounted(editor, monaco)
      if (this.diffEditor) {
        editor.onDidUpdateDiff(() => {
          this._emitChange(this._getValue(editor), event)
        })
      } else {
        editor.onDidChangeModelContent(event => {
          this._emitChange(this._getValue(editor), event)
        })
      }
    },
    _emitChange(value, event) {
      this.$emit('change', value, event)
      this.$emit('input', value)
    },
    _getValue(editor) {
      if (this.diffEditor) {
        return editor.getModel().modified.getValue()
      } else {
        return editor.getValue()
      }
    }
  },
  render(h) {
    return (
      <div class='monaco-editor-vue-container' style={this.style}></div>
    )
  }
}
</script>
<style lang="scss">
.monaco-editor-vue-container{
  border: 1px solid #7d7d7d;
}
</style>
