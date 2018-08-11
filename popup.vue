<!--
弹窗组件
例子暂可查看 home/base/data-area/intro-popup 文件 (根据需求使用以下几种示例, 事例文件使用第3种)

@author cui
-->
<!--
四种使用示例：
（推荐按以下模板使用，width可追加min/max-width, height相关推荐与模板一致，否则可能有样式问题）
1. 宽度650px,高度最大为浏览器高度80%，内容不足时自动变矮
  popup(
    :show.sync="show",
    popup-style="width: 650px",
    :percent-height=.8
  )
2. 宽度650px,高度最大为600px，内容不足时自动变矮(该方法使用最大高度推荐不超过650px)
  popup(
    :show.sync="show",
    popup-style="width: 650px; max-height: 600px",
  )
  2-1. 宽度650px,高度最大为屏幕高，内容不足自动变矮（尽可能完全显示数据 和 偷懒时使用）
    popup(
      :show.sync="show",
      popup-style="width: 650px,
    )
3. 宽度650px,高度为600px，盒子大小不会因内容多少产生变化(height必写，且不要使用min|max-height)
  popup(
    :show.sync="show",
    popup-style="width: 650px; height: 600px",
    fixed-size,
  )
4. 宽度650px,高度为浏览器高度60%，盒子大小不会因内容多少产生变化(不要写height)
  popup(
    :show.sync="show",
    popup-style="width: 650px",
    :percent-height=.6,
    fixed-size,
  )
 -->
<template lang="pug">
  .popup-container
    transition(name="fade")
      .popup-bg(
        v-if="showPopup",
        @click="dismissable && dismiss()"
      )
    transition(:name="animation")
      .popup-content(
        v-if="showPopup",
        :style="contentStyle",
      )
        .wrap
          //- 头部
          .popup-header(
            v-if="!hideHeader"
            ref="header",
          )
            slot(name="header")
            .close-button(
              v-if="!hideClose",
              @click="dismiss"
            )
          //- 内容区 （内容元素过少, 可能出现滚动条, 增加少量padding即可解决）
          .popup-body(
            ref="body",
            :style="'max-height:' + contentMaxHeight + 'px'",
            :class="{ 'hide-header': hideHeader, 'hide-footer': hideFooter }"
          )
            slot(name="body")
            .close-button(
              v-if="!hideClose && hideHeader",
              @click="dismiss"
            )

          //- 底部
          .popup-footer(
            v-if="!hideFooter"
            ref="footer",
            :class="{ fixedSize: fixedSize }"
          )
            slot(name="footer")
</template>

<script>
import { getElementProps } from 'common/js/util'

export default {
  props: {
    /**
     * 是否显示 popup
     */
    show: {
      default: false,
      type: Boolean,
    },

    /**
     * 作用于 popup-content 的额外样式
     */
    popupStyle: {
      type: String,
    },

    /* 自适应高度时，最大高度与浏览器窗口高度的比值（默认0关闭,推荐0.8）*/
    percentHeight: {
      type: Number,
      default: 0,
    },

    /**
     * 点击页面背景是否能关闭 popup
     */
    dismissable: {
      type: Boolean,
      default: true,
    },

    /**
     * 传入参数，是否使 popup 尺寸固定且不自适应(使用该属性需要自定义窗口高度样式，推荐不大于650px)
     */
    fixedSize: {
      type: Boolean,
      default: false,
    },

    /**
     * 隐藏头部
     */
    hideHeader: {
      type: Boolean,
      default: false,
    },

    /**
     * 隐藏底部
     */
    hideFooter: {
      type: Boolean,
      default: false,
    },

    /**
     * 隐藏关闭按钮
     */
    hideClose: {
      type: Boolean,
      default: false,
    },

    /**
     * 显隐时的动画，具体设置请参照 styles/animation.sass
     */
    animation: {
      type: String,
      default: 'fade-bottom',
    },
  },

  data () {
    return {
      // 复写 防止报错
      showPopup: this.show,

      contentMaxHeight: 0,
    }
  },

  mounted () {
    document.body.appendChild(this.$el)
  },

  beforeDestroy () {
    if (this.$el) this.$el.remove()
  },

  computed: {
    /**
     * 外部盒子样式
     */
    contentStyle () {
      if (this.percentHeight > 0) {
        const height = Math.floor(window.innerHeight * this.percentHeight)
        return this.fixedSize
          ? `height: ${height}px; ${this.popupStyle}`
          : `max-height: ${height}px; ${this.popupStyle}`
      } else {
        return this.popupStyle
      }
    },
  },
  watch: {
    show: {
      handler: function (val, oldVal) {
        if (val === oldVal) return
        this.showPopup = val

        // 这里处理了显示隐藏时滚动条相关
        if (this.showPopup) {
          // 现在页面宽度
          const nowWidth = document.documentElement.clientWidth
          document.documentElement.style.overflowY = 'hidden'

          // 禁止滚动时页面宽度
          const noScrollWidth = document.documentElement.clientWidth

          // 获取当前浏览器滚动条宽度
          // 如果原来没有滚动条，则两值相等，结果为0，无影响
          const scrollBarWidth = noScrollWidth - nowWidth
          document.documentElement.style.paddingRight = `${scrollBarWidth}px`
        } else {
          // 隐藏popup时重置受影响样式
          document.documentElement.style.overflowY = ''
          document.documentElement.style.paddingRight = ''
        }

        this.$nextTick(function () {
          // 重新计算 popup-body 最大高度
          this.contentMaxHeight = this.getContentMaxHeight()
        })
      },
      immediate: true,
    },
  },

  methods: {
    dismiss () {
      this.showPopup = false
      this.$emit('dismiss')
      this.$emit('update:show', false)
    },

    /**
     * 计算 popup-body 最大高度
     */
    getContentMaxHeight () {
      // 如果该元素 (header|footer) 没有设置隐藏, 且无法取得该元素dom,
      // 则容器不完整不需要计算高度, 且内部判断元素高度时会报错
      if (!this.hideHeader && !this.$refs.header) return 0
      if (!this.hideFooter && !this.$refs.footer) return 0

      // 如果隐藏头部，则不计算头部高度
      const headerHeight = !this.hideHeader
        ? getElementProps(this.$refs.header, 'height')
        : 0

      // 如果隐藏底部，则不计算底部高度
      const footerHeight = !this.hideFooter
        ? getElementProps(this.$refs.footer, 'height')
        : 0
      const height = +headerHeight + +footerHeight

      if (this.fixedSize && this.percentHeight <= 0) {
      // 如果固定盒子高度，则计算popup-body高度
        return this.popupStyle.match(/[^-]height:\s?(\d+)px/)[1] - height
      } else if (this.percentHeight > 0) {
      // 如果传入高度百分比，则按照百分比计算
        return Math.ceil(window.innerHeight * this.percentHeight) - height
      } else {
        // 未传入fixedHeight||percentHeight模式，则定义为标准样式。
        if ((/height/).test(this.popupStyle)) {
          return this.popupStyle.match(/height:\s?(\d+)px/)[1] - height
        } else {
          return window.innerHeight - 200
        }
      }
    },
  },
}
</script>

<style lang="sass" scoped>
  .popup-container
    position: fixed
    z-index: 1100
    .popup-bg
      position: fixed
      z-index: 1101
      left: 0
      right: 0
      top: 0
      bottom: 0
      background-color: rgba(0,0,0,.5)
    .popup-content
      padding: 0
      position: fixed
      top: 50%
      left: 50%
      transform: translate(-50%, -50%)
      z-index: 1102
      border-radius: 5px
      background-color: #FFF
      box-shadow: 0 0 5px rgba(0,0,0,.2)
      &.judgeSize
        max-height: auto

      .wrap
        height: 100%
        .popup-header
          min-height: 60px
          overflow: hidden
          position: relative
          border-radius: 5px 5px 0 0
          background-color: #fff
        .popup-body
          height: 100%
          position: relative
          overflow-y: auto
          word-break: break-word
          background-color: #fff
          &.hide-header
            border-top-left-radius: 5px
            border-top-right-radius: 5px
          &.hide-footer
            border-bottom-left-radius: 5px
            border-bottom-right-radius: 5px
        .popup-footer
          position: relative
          min-height: 60px
          background-color: #FFF
          border-radius: 0 0 5px 5px
          &.fixedSize
            position: absolute
            bottom: 0
            width: 100%

        .close-button
          position: absolute
          top: 14px
          right: 18px
          cursor: pointer
          width: 14px
          height: 14px
          background-size: 100%
          background-image: url(~@/assets/icon_guanbi.png)
</style>
