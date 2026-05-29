<template>
  <div class="main_app">
    <div class="app__main">
      <div class="app__content">
        <div class="app_top" />
        <div class="app_main">
          <div class="app_left" />
          <div class="app_center">
            <svg_editor />
          </div>
          <div class="app_right" />
        </div>
        <div class="app_bottom" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import svg_editor from './editor/main.vue'

import { defineComponent } from 'vue'

export default defineComponent({
  components: {
    svg_editor,
  },
  methods: {
    handleChange(event: any) {
      // `event` implicitly has `any` type
      console.log(event.target.value)
    },
  },
})
</script>

<style lang="scss">
.main_app {
  width: 100%;
  min-width: 1200px;
  min-height: 800px;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  box-sizing: border-box;
  padding: 50px;
  overflow: auto;

  > .app__main {
    position: relative;
    width: 100%;
    height: 100%;
    min-width: 1100px;
    min-height: 700px;
    background-color: rgb(197, 191, 187);

    > .app__content {
      width: calc(100% - 40px);
      height: calc(100% - 40px);
      margin: 20px;
      background-color: white;

      > .app_top {
        height: 100px;
        width: 100%;
        background-color: aliceblue;
      }
      > .app_main {
        height: calc(100% - 200px);
        width: 100%;
        background-color: rgb(130, 179, 222);

        > .app_left {
          width: 200px;
          height: 100%;
          background-color: rgb(255, 255, 255);
          float: left;
        }

        > .app_center {
          width: calc(100% - 400px);
          height: 100%;
          background-color: rgb(188, 174, 174);
          float: left;
        }

        > .app_right {
          width: 200px;
          height: 100%;
          background-color: rgb(255, 255, 255);
          float: right;
        }
      }
      > .app_bottom {
        height: 100px;
        width: 100%;
        background-color: aliceblue;
      }
    }
  }
}
</style>
