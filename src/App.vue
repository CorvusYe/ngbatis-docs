<script setup lang="ts">
import { computed, defineComponent, nextTick, onMounted, ref } from 'vue'
import { lang, setLang } from './language'
import MenuChild from './components/MenuChild.vue'
import axios from "axios"
import { useRouter, useRoute } from "vue-router";
import { currentVersion, widthLtHeight, setVersion } from './utils/window-util'
import { ElLoading } from 'element-plus'

import { useDark, useToggle, useNavigatorLanguage } from '@vueuse/core'
import { Moon, Sunny, Fold } from '@element-plus/icons-vue'

const languages = useNavigatorLanguage()
const isDark = useDark()
const toggleDark = useToggle(isDark)
const isSunny = ref(!isDark.value)
const showMenu = ref(false)

function turnLight() {
  isDark.value = !isSunny.value
}
const prefLang = ref(lang)


let router = useRouter()
let route = useRoute()
let menus = ref([])
let defaultOpeneds = ref([])
let menu = ref(undefined as any)
let pathId = ref(undefined)
let menuId = ref(undefined)

const versions = ref([])
const version = ref(currentVersion())

defineComponent({
  name: 'app',
  components: { MenuChild }
})

function switchLang() {
  setLang(prefLang.value);
  forceGo();
}

function switchVersion() {
  setVersion(version.value)
  forceGo();
}

async function getMenu() {
  const res = await axios({
    method: 'get',
    url: `./docs/${version.value}/${lang}/nav.json`
  })

  menus.value = await res.data
  let pathAndFile = getPathAndFile(menus.value)
  if( pathAndFile )
  menu.value = findMenu(menus.value, pathAndFile)
}

function getPathAndFile(menus) {
  let path , file;
  if( route.query.path && route.query.file ) {
    path = route.query.path;
    file = route.query.file;
    return [ path, file ]
  } else {
    let traceStr = sessionStorage.getItem('menuIndex')
    if( traceStr ) {
      let trace = JSON.parse(traceStr)
      toPage(trace[1], trace)
      return
    }
    let m = menus[0].children[0]
    toPage(m, [menus[0], m])
  }
}

function findMenu(menus, ids) {
  let id = ids.shift()
  for( let i = 0; i < menus.length; i ++ ) {
    let m = menus[i]
    if( m.id == id ) {
      if(ids.length) {
        defaultOpeneds.value.push(m);
        return findMenu(m.children, ids)
      }
      return m
    }
  }
}

async function getVersions() {
  const loadingInstance1 = ElLoading.service({ fullscreen: false, target: '.j_doc' })
  try {
    const res = await axios({
      method: 'get',
      url: `./docs/versions.json`
    })
    versions.value = await res.data
  } catch (e) {}
  loadingInstance1.close()
}

function toPage(menu: any, trace: Array<any>) {
  showMenu.value = false
  sessionStorage.setItem('menuIndex', JSON.stringify(trace))
  pathId.value = trace[0].id
  menuId.value = menu.id
  go()
}

function go() {
  router.push({
    path: `/`,
    query: {
      path: pathId.value,
      file: menuId.value,
      v: version.value
    }
  });
}

function forceGo() {
  location.hash = `#/?path=${pathId.value}&file=${menuId.value}&v=${version.value}`
  location.reload()
}

getMenu()
const isVertial = ref(false)
nextTick(() => {
  isVertial.value = widthLtHeight()
})

getVersions();

</script>

<template>
  <el-container>
    <el-header>
      <el-row>
        <el-col :span="14">
          <span ><a href="https://github.com/CorvusYe/ngbatis-docs" target="_blank">Ngbatis</a></span> 
          <span v-if="!isVertial" class="desc">
            <el-divider direction="vertical"/>{{$t('desc')}}
          </span>
        </el-col>
        <el-col :span="10" style="text-align: right !important;">
          <el-switch v-model="isSunny"
            :active-icon="Sunny"
            :inactive-icon="Moon"
            @change="turnLight"
            inline-prompt
          ></el-switch>
          <el-switch 
            v-model="prefLang"
            active-value="zhCn"
            inactive-value="en"
            active-text="En"
            inline-prompt
            style="--el-switch-on-color: var(--el-color-primary); --el-switch-off-color: var(--el-color-primary)"
            inactive-text="???"
            @change="switchLang"></el-switch>
        </el-col>
      </el-row>
    </el-header>
    <el-container>
      <el-aside width="300px" v-if="!isVertial">
        <el-menu @select="toPage" 
          :default-active="menu" 
          :default-openeds="defaultOpeneds">
          <menu-child :menus="menus"></menu-child>
        </el-menu>
        <center style="color: var(--el-color-info)">
          <el-select v-model="version" 
              @change="switchVersion"
              size="small">
            <el-option 
              v-for="v in versions"
              :key="v.version"
              :label="v.version"
              :value="v.version">
              <span>{{v.version}}</span><el-icon><Upload/></el-icon>
            </el-option>
          </el-select>
        </center>
      </el-aside>
      <el-drawer size="70%" v-model="showMenu" direction="ltr" :show-close="false">
        <el-menu @select="toPage" 
          :default-active="menu" 
          :default-openeds="defaultOpeneds">
          <menu-child :menus="menus"></menu-child>
        </el-menu>
        <center style="color: var(--el-color-info)">
          <el-select v-model="version" 
              @change="switchVersion"
               size="small">
            <el-option 
              v-for="v in versions"
              :key="v.version"
              :label="v.version"
              :value="v.version">
            </el-option>
          </el-select>
        </center>
      </el-drawer>
      <el-main width="100%" :class="{ isVertical: isVertical }">
        <router-view :key="$route.fullPath"/>
        <div class="copy-right">
          <span>{{$t('copyRight')}}</span>
          <span><a href="https://github.com/nebula-contrib" target="_blank">{{$t('organization')}}</a></span>
          <span> & </span>
          <span><a href="https://github.com/CorvusYe" target="_blank">{{$t('author')}}</a></span>
        </div>
      </el-main>
    </el-container>
  </el-container>
  <el-button v-if="isVertial" type="primary" :icon="Fold" circle  style="position: fixed; right: 30px; bottom: 60px; z-index: 9999;" @click="showMenu = true"/>
</template>

<style scoped>
#app .el-header {
  line-height: 52px;
  font-size: 28px;
  padding: 0 16px;
  border-bottom: 1px solid var(--el-border-color);
  flex-direction: row;
}

#app .el-header .desc {
  font-size: 16px;
  text-overflow: ellipsis;
  max-lines: 1;
}
#app .el-container .el-aside .el-menu {
  height: calc( 100vh - 90px );
}

#app .el-container .copy-right {
  line-height: 30px;
  text-align: center;
  color: var(--el-color-info-light-3) !important;
}
.copy-right a[data-v-7a7a37b1], .copy-right a[data-v-7a7a37b1]:link, .copy-right a[data-v-7a7a37b1]:visited, .copy-right a[data-v-7a7a37b1]:hover, .copy-right a[data-v-7a7a37b1]:active{
  color: var(--el-color-info-light-3) !important;
}

#app .el-container {
  height: 100vh;
  width: 100vw;
}
#app .el-container .el-container {
  height: calc( 100vh - 60px );
}
#app .el-main {
  padding: 0;
  height: calc( 100vh - 60px );
}
.el-switch {
  padding: 0px 8px;
}
.el-drawer  .el-drawer__body {
  padding: 0 !important;
}
#app .el-drawer .el-menu {
  border: none;
}

a, a:link, a:visited, a:hover, a:active {
    text-decoration: none;
    color: var(--el-color-primary) !important;
}
</style>
