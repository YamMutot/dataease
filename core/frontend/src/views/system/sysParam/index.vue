<template>
  <de-layout-content
    v-loading="$store.getters.loadingMap[$store.getters.currentPath]"
  >
    <div class="sys-setting">
      <el-tabs
        v-model="activeName"
        class="de-tabs"
      >
        <el-tab-pane
          lazy
          :label="$t('system_parameter_setting.basic_setting')"
          name="zero"
        />

        <el-tab-pane
          lazy
          :label="$t('system_parameter_setting.mailbox_service_settings')"
          name="first"
        />

        <el-tab-pane
          lazy
          :label="$t('sysParams.map')"
          name="ten"
        />

        <el-tab-pane
          v-if="engineMode === 'simple'"
          lazy
          :label="$t('system_parameter_setting.engine_mode_setting')"
          name="six"
        />

        <el-tab-pane
          v-if="engineMode === 'cluster'"
          lazy
          :label="$t('system_parameter_setting.engine_mode_setting')"
          name="seven"
        />

        <el-tab-pane
          v-if="engineMode === 'cluster'"
          lazy
          :label="$t('system_parameter_setting.kettle_setting')"
          name="eight"
        />

        <el-tab-pane
          v-if="isPluginLoaded && showProxy"
          lazy
          :label="$t('system_parameter_setting.proxy_setting')"
          name="nine"
        />
      </el-tabs>
      <div
        class="tabs-container"
        :class="[activeName !== 'eight' ? 'is-center' : 'pad-center']"
      >
        <div :class="activeName === 'ten' ? 'max-w600' : 'min-w600'">
          <basic-setting
            v-if="activeName === 'zero'"
            :is-plugin-loaded="isPluginLoaded"
          />
          <email-setting v-if="activeName === 'first'" />
          <map-setting v-if="activeName === 'ten'" />

          <simple-mode v-if="activeName === 'six'" />
          <cluster-mode v-if="activeName === 'seven'" />
          <kettle-setting v-if="activeName === 'eight'" />
          <plugin-com
            v-if="isPluginLoaded && activeName === 'nine'"
            ref="ProxySetting"
            component-name="ProxySetting"
          />
        </div>
      </div>
    </div>
  </de-layout-content>
</template>
<script>
import BasicSetting from './BasicSetting'
import MapSetting from './MapSetting'
import EmailSetting from './EmailSetting'
import SimpleMode from './SimpleModeSetting'
import ClusterMode from './ClusterModeSetting'
import KettleSetting from './KettleSetting'
import DeLayoutContent from '@/components/business/DeLayoutContent'
import { pluginLoaded } from '@/api/user'
import { engineMode } from '@/api/system/engine'
import PluginCom from '@/views/system/plugin/PluginCom'
export default {
  components: {
    BasicSetting,
    EmailSetting,
    DeLayoutContent,
    SimpleMode,
    ClusterMode,
    KettleSetting,
    MapSetting,
    PluginCom
  },
  data() {
    return {
      activeName: 'zero',
      isPluginLoaded: false,
      engineMode: 'local',
      showProxy: false
    }
  },
  beforeCreate() {
    pluginLoaded().then((res) => {
      this.isPluginLoaded = res.success && res.data
    })
    engineMode().then((res) => {
      this.engineMode = res.data
    })
  }
}
</script>
<style lang="scss" scoped>
.sys-setting {
  height: 100%;
  background-color: var(--MainBG, #f5f6f7);

  .tabs-container {
    height: calc(100% - 48px);
    background: var(--ContentBG, #ffffff);
    overflow-x: auto;

    .min-w600 {
      min-width: 600px;
      height: 100%;
      & > :nth-child(1) {
        box-sizing: border-box;
        padding-bottom: 20px;
      }
    }
    .max-w600 {
      width: 100%;
      height: 100%;
      & > :nth-child(1) {
        box-sizing: border-box;
        padding-bottom: 20px;
      }
    }
  }

  .is-center {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0 20px 20px 20px;
  }

  .pad-center {
    padding: 24px;
  }
}
</style>
